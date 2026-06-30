# Phase 7 设计：手机控制类工具

> 本文档定义 Phase 7 的 4 个手机控制工具设计。  
> 阶段：设计草案（待 Phase 1-6 完成后实施）  
> 依赖：Phase 6（联网工具调用，提供 Tool 接口 + 状态机扩展）+ Phase 5（发布）完成  
> 与主设计关系：[00_design_overview.md](./00_design_overview.md) §11 路线图的扩展

---

## 0. 设计目标

利用 root + HyperOS 提供本地系统控制能力，让 MiBrain 能"动手机"。4 个工具按稳定性排序：

| Tool | 做什么 | root 价值 | 工程量 |
|---|---|---|---|
| SystemSettingsTool | 切换手电筒/亮度/静音等 | 部分 root，部分免 root | 小 |
| NotificationReaderTool | 朗读通知 + 语音回复 | root 自动授权通知访问 | 中 |
| AppLauncherTool (L1) | `am start` 启动 app | root 跳过权限 | 小 |
| ScreenCaptureTool | 截屏 + 云端 vision 看屏 | root 一次授权后免再确认 | 中 |

---

## 1. SystemSettingsTool（最稳，推荐先做）

### 1.1 支持的设置项

| 设置 | 实现方式 | 是否需 root | 备注 |
|---|---|---|---|
| 手电筒 | `CameraManager.setTorchMode()` | 否 | API 23+ |
| 屏幕亮度 | `Settings.System.SCREEN_BRIGHTNESS` | root 强制授 WRITE_SETTINGS | 0-255 |
| 自动亮度开关 | `Settings.System.SCREEN_BRIGHTNESS_MODE` | 同上 | 0=手动 / 1=自动 |
| 静音/响铃/震动 | `AudioManager.ringerMode` | 否 | |
| 媒体音量 | `AudioManager.setStreamVolume(STREAM_MUSIC)` | 否 | |
| 闹钟音量 | 同上 STREAM_ALARM | 否 | |
| 飞行模式 | `cmd connectivity airplane-mode enable/disable` | 是 | HyperOS 行为待测 |
| 热点 | `cmd connectivity start-softap` | 是 | HyperOS 行为待测 |
| 蓝牙开关 | `cmd bluetooth enable/disable` | 是 | |
| Wi-Fi 开关 | `cmd wifi set-wifi-enabled` | 是 | |
| 自动旋转 | `Settings.System.ACCELEROMETER_ROTATION` | 同亮度 | 0=关 / 1=开 |

### 1.2 关键词路由

| 用户说 | 路由到 |
|---|---|
| "开/关手电筒" | torch on/off |
| "调亮/调暗屏幕" | brightness up/down |
| "静音/响铃" | ringer silent/normal |
| "调大/调小音量" | volume up/down |
| "开/关飞行模式" | airplane on/off |
| "开/关热点" | hotspot on/off |
| "开/关蓝牙/Wi-Fi" | bluetooth/wifi on/off |
| "开/关自动旋转" | autorotate on/off |

### 1.3 失败处理

- 免 root 设置项失败 → 提示"设置失败，请检查权限"
- root 命令失败 → 检查 `cmd` 是否存在（HyperOS 路径可能变），失败时记录日志
- 用户说"调亮屏幕"但当前已是最大亮度 → 提示"屏幕已是最亮"

---

## 2. NotificationReaderTool（杀手场景）

### 2.1 朗读流程

```
微信来通知
  ↓
NotificationListenerService.onNotificationPosted
  ↓
提取 title="妈妈", text="我到了"
  ↓
检查 Settings.notificationReaderEnabled
  ↓
检查当前 state == IDLE（不打断 SPEAKING/LISTENING）
  ↓
state = SPEAKING
  ↓
TTS 播报"妈妈发来微信：我到了"
  ↓
state = COOLDOWN → IDLE
```

### 2.2 支持的 app（白名单）

| App | 包名 | 朗读格式 | 是否支持回复 |
|---|---|---|---|
| 微信 | com.tencent.mm | "{title}发来微信：{text}" | 是（remoteInput） |
| QQ | com.tencent.mobileqq | "{title}发来QQ：{text}" | 是（remoteInput） |
| 短信 | com.android.messaging | "短信来自{title}：{text}" | 是 |
| 来电 | com.android.phone | "来电：{title}"，朗读后等待用户说"接听/挂断" | 接听/挂断 |
| 其他 app | 默认 | "{appName}通知：{title} - {text}" | 否 |

**默认白名单**：微信 + QQ + 短信 + 来电 + 其他所有 app（用户可在 Settings 里关"其他 app"）。

### 2.3 语音回复

```
用户："回复好的马上来"
  ↓
ASR → "好的马上来"
  ↓
检查最近 60s 内是否有可回复通知
  ↓
找到通知的 remoteInput Action
  ↓
填入 "好的马上来" 触发 Action
  ↓
TTS："已回复妈妈：好的马上来"
```

**风险**：微信版本更新可能破坏 remoteInput Action 的可用性。Phase 7 实施时需测试当前微信版本。

### 2.4 root 价值

`NotificationListenerService` 本身免 root，但用户需要去"设置 → 通知访问"里手动授权。root + KSU `appops set <uid> POST_NOTIFICATIONS allow` 可自动授权。

### 2.5 状态机交互

- IDLE 状态下通知到达 → 直接朗读
- LISTENING/THINKING/SPEAKING 状态下通知到达 → 排队，等回到 IDLE 后再朗读（避免打断当前对话）
- 排队上限 3 条，超过的丢弃（避免用户长时间不在被打扰）

---

## 3. AppLauncherTool（L1：仅启动 app）

### 3.1 实现

```kotlin
fun launch(appName: String): Boolean {
    val pkg = resolvePackageName(appName)  // 从已装 app 列表匹配
    if (pkg == null) {
        tts.speak("没找到 $appName 这个应用")
        return false
    }
    val cmd = "am start -n $pkg/.MainActivity"  // 或 monkey 启动
    Shell.exec(cmd)  // root 调用
    tts.speak("已打开 $appName")
    return true
}
```

### 3.2 应用名匹配

维护一个"中文常用名 → 包名"映射表，覆盖 Top 50 应用：

| 中文常用名 | 包名 |
|---|---|
| 微信 | com.tencent.mm |
| QQ | com.tencent.mobileqq |
| 抖音 | com.ss.android.ugc.aweme |
| 支付宝 | com.eg.android.AlipayGphone |
| 淘宝 | com.taobao.taobao |
| 京东 | com.jingdong.app.mall |
| 网易云音乐 | com.netease.cloudmusic |
| ... | ... |

未命中映射表时，遍历 `PackageManager.getInstalledApplications()` 做 fuzzy match（包含匹配）。

### 3.3 范围明确

L1 仅做启动，**不做**：
- ❌ 跳转到特定页面（L2，留后续）
- ❌ 填文本（L3）
- ❌ 自动点击发送（L4）

---

## 4. ScreenCaptureTool（截屏 + vision）

### 4.1 流程

```
用户："屏幕上说的是什么意思"
  ↓
ASR → 文本
  ↓
ToolRouter → ScreenCaptureTool
  ↓
NetworkGate check (联网开关必须开)
  ↓
state = TOOL_RUNNING
  ↓
TTS："好的，正在截屏"
  ↓
root 截屏：screencap -p /sdcard/mibrain_capture.png
  ↓
上传到 vision API（默认 GLM-4V，用户可改 GPT-4V/Claude）
  ↓
vision API 返回描述
  ↓
LLM 组织语言
  ↓
TTS："屏幕上显示的是..."
```

### 4.2 Vision API 选型

| API | 优势 | 价格 | 默认 |
|---|---|---|---|
| GLM-4V（智谱） | 国内访问稳，中文好 | 收费 | ✅ 默认 |
| GPT-4V | 质量最好 | 收费 | 备选 |
| Claude 3.5 Sonnet | 长文好 | 收费 | 备选 |
| Qwen-VL | 国产，部分可本地 | 部分免费 | 备选 |

**用户需配置 vision API key**，未配置时禁用此工具。

### 4.3 隐私告知（强制 UI 提示）

首次启用此工具时，强制弹出对话框：
> "屏幕截屏工具会将手机屏幕截图发送到 [API 名称] 进行识别。这会泄漏屏幕上的任何信息（包括聊天、密码框外的内容）。是否继续？"

用户确认后才启用。

### 4.4 截屏权限

- root 模式：`screencap -p` 直接截屏，无需每次授权
- 免 root 模式：`MediaProjection` API，每次启动需用户授权一次（前台服务期间保持）

### 4.5 状态机

依赖 Phase 6 的 `TOOL_RUNNING` 状态，无需新增。

---

## 5. UI 设计

```
Settings → 手机控制
├── [总开关] 启用手机控制      ▢
├── SystemSettingsTool
│   ├── [开关] 允许切换系统设置   ▢
│   └── [开关] 允许 root 命令（飞行模式/热点等）  ▢
├── NotificationReaderTool
│   ├── [开关] 启用通知朗读      ▢
│   ├── [开关] 允许语音回复      ▢
│   ├── 朗读来源
│   │   ├── □ 微信
│   │   ├── □ QQ
│   │   ├── □ 短信
│   │   ├── □ 来电
│   │   └── □ 其他所有 app
│   └── 排队上限: 3
├── AppLauncherTool
│   └── [开关] 允许启动应用     ▢
└── ScreenCaptureTool
    ├── [开关] 启用截屏看屏     ▢
    ├── Vision API
    │   ├── 提供商: [GLM-4V ▼]
    │   └── API Key: [______]
    └── □ 我已知晓隐私风险
```

---

## 6. 数据流时序（一次通知朗读 + 回复）

```
T=0   微信来通知："妈妈：我到了"
T=0.1 NotificationListenerService.onNotificationPosted
T=0.1 check state == IDLE → OK
T=0.1 state = SPEAKING
T=0.2 TTS："妈妈发来微信：我到了"
T=2   播放完毕
T=2.5 state = COOLDOWN → IDLE
T=5   用户："回复好的马上来"
T=6   ASR → "好的马上来"
T=6   找最近 60s 通知的 remoteInput Action
T=6.5 填入 "好的马上来" 触发 Action
T=7   TTS："已回复妈妈：好的马上来"
T=9   state = IDLE
```

总延迟 ~2s 朗读 + ~2s 回复确认。

---

## 7. 工程量估算

| 模块 | 行数估算 | 备注 |
|---|---|---|
| SystemSettingsTool | 400 | 10+ 设置项 × 40 行 |
| NotificationReaderTool | 600 | 朗读 200 + 回复 300 + UI 100 |
| AppLauncherTool (L1) | 200 | 启动 + 应用名映射表 |
| ScreenCaptureTool | 500 | 截屏 100 + vision API 200 + 隐私 UI 100 + 状态机集成 100 |
| UI 设置页 | 300 | Compose |
| 测试 + 集成 | 300 | |
| **合计** | **~2300 行 Kotlin** | 约为 Phase 6 + Phase 2 之和 |

---

## 8. 范围外（明确不做）

- ❌ AppLauncherTool 的 L2/L3/L4（留后续 Phase 或用户贡献插件）
- ❌ 自动接听电话（需要 hook TelecomManager，复杂）
- ❌ 来电拦截/挂断（需 root + TelephonyManager hook）
- ❌ 短信自动回复（NotificationReaderTool 已覆盖，不单独做）
- ❌ 多 vision API fallback（用户自己改 key）
- ❌ 截屏历史记录（每次截屏即用即弃）
- ❌ 智能家居控制（米家 hook 太复杂，留 Phase 8 或不做）

---

## 9. 风险与缓解

| 风险 | 概率 | 影响 | 缓解 |
|---|---|---|---|
| 微信 remoteInput Action 在新版被禁 | 高 | 中 | Phase 7 实施时测试当前微信版本；不可用时降级为"只朗读，不回复" |
| HyperOS 改 `cmd` 命令路径 | 中 | 中 | 在 SystemSettingsTool 里做 fallback：先试 `cmd`，失败用 `settings`/`service` |
| 截屏发云端泄漏隐私 | 中 | 高 | 强制 UI 确认 + 默认关闭 + 默认每次截屏前再 TTS 提示"正在截屏" |
| 通知朗读打断正在进行的对话 | 中 | 低 | 排队机制（最多 3 条），SPEAKING 期间不朗读 |
| 来电朗读时被对方听见 | 低 | 中 | 来电时不朗读，等响铃 3 次后或用户接听后才朗读来电人 |

---

## 10. 与 Phase 6 的关系

- 共用 `Tool` 接口与 `ToolRouter`
- 共用 `TOOL_RUNNING` 状态
- 共用 UI 设置页框架
- NotificationReaderTool 是**唯一一个不通过 ToolRouter 触发**的工具——它是事件驱动（通知到达），不是用户请求驱动
- ScreenCaptureTool 强依赖 Phase 6 的联网开关

---

## 11. 待 Phase 6 实施前再确认的开放问题

| # | 问题 | 当前默认 |
|---|---|---|
| Q1 | 微信当前版本是否还暴露 remoteInput Action？ | 假设有，Phase 7 实施时实测 |
| Q2 | HyperOS 上 `cmd connectivity start-softap` 是否有效？ | 假设有，Phase 7 实施时实测 |
| Q3 | GLM-4V API 是否仍可用且价格可接受？ | 假设有，Phase 7 实施前复查 |
| Q4 | 来电朗读的时机如何把握（响铃几次后/接听后/挂断后）？ | 默认响铃 3 次后朗读 |
| Q5 | 是否需要为 NotificationReaderTool 单独的"勿扰时段"设置？ | 暂不做，用户可手动关闭 |
