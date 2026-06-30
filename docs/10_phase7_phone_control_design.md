# Phase 7 设计：手机控制类工具

> 本文档定义 Phase 7 的 4 个手机控制工具设计。  
> 阶段：设计草案（待 Phase 1-6 完成后实施）  
> 依赖：Phase 6（联网工具调用，提供 Tool 接口 + 状态机扩展）+ Phase 5（发布）完成  
> 与主设计关系：[00_design_overview.md](./00_design_overview.md) §11 路线图的扩展

> **2026-06-30 修订说明**（深度检查后第二轮调整）：
> - **S5 修复**：原 §2.4 `appops set POST_NOTIFICATIONS allow` 是错误判断——appops 无法授权 NotificationListenerService，应改为 `settings put secure enabled_notification_listeners` 或明示需用户手动授权
> - **S6 修复**：微信通知回复路径改 A2A 协议（2026-06 起微信推进 A2A，Wear RemoteInput 逐步失效），详见 §2.3
> - **M1 修复**：AppLauncherTool 改用 `monkey -p $pkg 1`（[D19](../DECISIONS.md) L1 仅启动 app）
> - 推理后端切回 JNI（[D7](../DECISIONS.md)）
> - 唤醒词改 sherpa-onnx KWS（[D23](../DECISIONS.md)）

---

## 0. 设计目标

利用 root + HyperOS 提供本地系统控制能力，让 MiBrain 能"动手机"。4 个工具按稳定性排序：

| Tool | 做什么 | root 价值 | 工程量 |
|---|---|---|---|
| SystemSettingsTool | 切换手电筒/亮度/静音等 | 部分 root，部分免 root | 小 |
| NotificationReaderTool | 朗读通知 + 语音回复 | root 部分自动授权（详见 §2.4） | 中 |
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

### 2.3 语音回复（S6 修复：A2A 路线）

⚠️ **2026-06 起微信推进 A2A 协议**，原有的 Wear RemoteInput（RemoteInput + WearNotificationExtender 模拟手表回复）路径**逐步失效**。需要切到 A2A 路线或微信开放平台 API。
本项目 Phase 7 的微信通知回复能力**降级为实验性**，需 Phase 7 真机验证：
- 短期（Phase 7 MVP）：保留 Wear RemoteInput 作为兜底（如果仍可用）
- 中期（Phase 7+）：调研 A2A 协议或微信开放平台 API 接入
- 长期：跟随上游微信协议演进

#### 2.3.1 短期兜底方案：Wear RemoteInput（原方案）

```
用户："回复好的马上来"
  ↓
ASR → "好的马上来"
  ↓
检查最近 60s 内是否有可回复通知
  ↓
找到通知的 remoteInput Action（RemoteInput + WearNotificationExtender 模拟手表回复）
  ↓
填入 "好的马上来" 触发 Action
  ↓
TTS："已回复妈妈：好的马上来"
```

**风险**：微信版本更新可能破坏 remoteInput Action 的可用性。Phase 7 实施时需测试当前微信版本。2026-06 起随 A2A 推进，此路径预期逐步失效，仅作为短期兜底。

#### 2.3.2 中期方案：A2A 协议（已核实不对第三方 APK 开放，移除该路径）

> ⚠️ **2026-06-30 第三轮核实**（[E9](./14_feasibility_recheck_and_plan.md) + [D29](../DECISIONS.md)）：原稿"调研 A2A 是否对第三方 APK 开放"已通过 web 核实得出**明确否定结论**：
> - 微信 2026-06-04 与华为、荣耀、小米、OPPO、vivo 合作推出 A2A 助手能力（[hellochinatech.com](https://hellochinatech.com/p/wechat-ai-gatekeeper)、[itbear.com](https://www.itbear.com/technews/wechat-teams-up-with-phone-makers-to-let-ai-assistants-send-messages-and-start-calls-by-voice/)）
> - 荣耀 YOYO 首批 2026-06-10 上线（Magic8/500/X70 系列）
> - **A2A 是腾讯控制的访问机制**，不是 Google/Linux Foundation 开放的 Agent2Agent 协议
> - **关键事实**：A2A 仅对**手机系统级 AI 助手**（华为 Xiaoai / 小米 XiaoAi / 荣耀 YOYO / OPPO / vivo）开放，**第三方 APK（如 MiBrain）不可直接接入**
> - 当前能力范围窄：仅支持发消息 + 语音/视频通话，红包/转账/朋友圈不支持
> - 决策权始终在微信侧，腾讯可随时扩缩权限

**结论**：MiBrain 作为第三方 APK，**A2A 路径不通**。原中期方案移除。

**新中期方案**：通知朗读 + 引导用户手动回复
- 朗读通知后，TTS 提示"如需回复，请打开微信手动操作"
- Phase 7+ 不再调研 A2A 接入（已核实不可行）
- 若用户确实需要语音回复微信，建议改用系统级 AI 助手（小爱同学 HyperOS 3 版本预计将接入微信 A2A，可直接用小爱替代 MiBrain 的微信回复场景）

### 2.4 通知权限（S5 修复：appops 无法授权 NotificationListenerService）

⚠️ **S5 修复（2026-06-30）**：原方案"`appops set <uid> POST_NOTIFICATIONS allow` 自动授权 NotificationListenerService"是错误判断，详见顶部修订说明。

**正确认知**：通知相关存在两种独立权限，授权路径不同：

| 权限 | 用途 | 存储位置 | root 能否自动 |
|---|---|---|---|
| POST_NOTIFICATIONS（Android 13+ 运行时权限） | app 自身在前台显示通知 | appops | ✅ 可，KSU `post-fs-data.sh` 用 `appops set <uid> OP_POST_NOTIFICATIONS allow` 自动授权 |
| NotificationListenerService 通知使用权 | 读取其他 app 的通知 | `Settings.Secure.ENABLED_NOTIFICATION_LISTENERS` | ❌ appops 无此能力，必须用户手动或 root 强制写 settings |

**推荐方案（首选）**：APK 引导用户手动授权
- 首次启用 NotificationReaderTool 时，弹出对话框引导用户去"设置 → 通知 → 通知使用权 → MiBrain"
- 检测状态：读 `Settings.Secure.getString(contentResolver, "enabled_notification_listeners")`，判断是否包含 `com.mibrain/.service.MiBrainNotificationListener`
- 未授权时禁用朗读功能 + TTS 提示"请先在系统设置里授予 MiBrain 通知使用权"

**Fallback（root 强制，有副作用，仅高级用户）**：

```bash
# KSU post-fs-data.sh 或手动执行
settings put secure enabled_notification_listeners com.mibrain/.service.MiBrainNotificationListener
# 需重启 systemui 让 NotificationManagerService 重新读取
pkill systemui
# 或：am force-stop com.android.systemui
```

- ⚠️ 副作用：systemui 重启会闪一下状态栏；锁屏前执行可能失败；部分 HyperOS 版本会在下次开机时回滚此项
- 不作为默认路径，仅作为 power user 的 escape hatch

#### 2.4.1 敏感通知朗读风险（R10/R11 第三轮核实新增）

> ⚠️ **R10 + R11 第三轮 web 核实新增**（[R10](./14_feasibility_recheck_and_plan.md) + [R11](./14_feasibility_recheck_and_plan.md)）：原 §2.2 通知白名单未考虑敏感信息（验证码 / 银行短信）的泄露风险，TTS 朗读会把明文通知播报到任何在场的人听，需补充过滤策略。

**风险场景**：

| 风险 ID | 场景 | 影响 | 缓解策略 |
|---|---|---|---|
| R10 | **验证码短信被朗读**：用户在公共场合，银行/微信/支付宝验证码短信到达，TTS 把 6 位验证码明文朗读出来 | 高（验证码泄露，他人可盗刷） | ① 默认识别 6 位数字短信，识别到验证码模式 → 仅 TTS 播报"你有一条验证码短信，请查看手机"，**不朗读验证码明文**；② 提供开关允许用户主动选择朗读明文 |
| R11 | **银行/支付通知被朗读**：余额变动、转账短信、信用卡账单通知到达 | 中（财务信息泄露） | ① 默认对"银行"、"转账"、"余额"等关键词做敏感过滤；② 默认仅朗读 "{appName} 通知：您有一条新消息"，不朗读 text；③ 提供开关允许用户主动选择完整朗读 |
| R10+ | **锁屏状态被他人触发**：锁屏后通知仍会触发朗读，他人可通过发短信窃取验证码 | 高（社会工程攻击） | ① 锁屏状态下验证码短信默认不朗读，仅记录到通知历史；② 解锁后用户主动查询时再朗读；③ 提供"勿扰时段"开关，夜间不朗读 |

**默认白名单升级**（基于 §2.2 原表）：

| App | 包名 | 朗读格式 | 默认处理 |
|---|---|---|---|
| 微信 | com.tencent.mm | "{title}发来微信：{text}" | 完整朗读（默认信任） |
| QQ | com.tencent.mobileqq | "{title}发来QQ：{text}" | 完整朗读（默认信任） |
| 短信 | com.android.messaging | "短信来自{title}：{text}" | **R10 默认敏感过滤**：识别 6 位数字验证码模式 → 仅"收到验证码短信，请查看手机"；识别"银行/转账/余额" → "收到银行短信，请查看手机" |
| 来电 | com.android.phone | "来电：{title}"，朗读后等待用户说"接听/挂断" | 不变 |
| 银行 app | com.*.bank.* | "{appName} 通知：您有一条新消息" | **R11 默认仅朗读 appName**，不朗读 text；用户可在 Settings 里改"完整朗读" |
| 支付宝 | com.eg.android.AlipayGphone | "{appName} 通知：您有一条新消息" | **R11 默认仅朗读 appName** |
| 其他 app | 默认 | "{appName}通知：{title} - {text}" | 完整朗读（默认信任） |

**UI 设置升级**（基于 §5 原图）：
- 朗读来源 增加 "□ 敏感过滤（验证码/银行/支付仅播报收到，不朗读明文）"，默认 ✅
- 新增 "□ 锁屏状态不朗读敏感通知"，默认 ✅
- 新增 "□ 勿扰时段（如 22:00-07:00 不朗读任何通知）"，默认 ❌

**实施细节**：
- 验证码识别正则：`/(?:验证码|校验码|verification code)[：:\s]*(\d{4,8})/i` + `/(?:^|\D)(\d{6})(?:\D|$)/`（兜底匹配连续 6 位数字）
- 敏感关键词黑名单：银行/转账/余额/支付/到账/消费/信用卡/账单/还款/逾期
- 关键词命中 → 仅朗读 appName，不朗读 text
- 用户主动查询时（"刚才那条短信是什么"），解锁后完整朗读

### 2.5 状态机交互

- IDLE 状态下通知到达 → 直接朗读
- LISTENING/THINKING/SPEAKING 状态下通知到达 → 排队，等回到 IDLE 后再朗读（避免打断当前对话）
- 排队上限 3 条，超过的丢弃（避免用户长时间不在被打扰）

---

## 3. AppLauncherTool（L1：仅启动 app）

### 3.1 实现（M1 修复：改用 `monkey -p`）

⚠️ **M1 修复（2026-06-30）**：原方案 `am start -n $pkg/.MainActivity` 需预知 LAUNCHER activity 名，匹配困难。改用 `monkey -p $pkg 1`（[D19](../DECISIONS.md) L1 仅启动 app），monkey 自动找 LAUNCHER activity。

```kotlin
class AppLauncherTool {
    fun launch(appName: String): Boolean {
        val pkg = resolvePackageName(appName)  // 从已装 app 列表匹配
        if (pkg == null) {
            tts.speak("没找到 $appName 这个应用")
            return false
        }
        // M1 修复：monkey -p 自动找 LAUNCHER activity，不需预知 activity 名
        val cmd = "monkey -p $pkg 1"
        val result = SuShell.execute(cmd)  // root 调用
        val ok = result.success && result.output.contains("Events injected")
        if (ok) {
            tts.speak("已打开 $appName")
        } else {
            tts.speak("打开 $appName 失败")
        }
        return ok
    }
}
```

**优点**：不需要知道具体 activity 名（monkey 自动找 LAUNCHER activity）。
**缺点**：monkey 命令输出可能有噪声，需用 `Events injected` 字符串判定成功。
**替代方案**：`am start -a android.intent.action.MAIN -c android.intent.category.LAUNCHER -n $pkg/$launcher_activity`，但需先 query LAUNCHER activity，工程量更大，L1 不采用。

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

> 以下时序对应 §2.3.1 短期兜底方案（Wear RemoteInput）。若 Wear RemoteInput 已失效（详见 §2.3 S6 风险），回复环节降级为"只朗读，不回复"。

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
T=6   找最近 60s 通知的 remoteInput Action（Wear RemoteInput 短期兜底）
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
| 微信 Wear RemoteInput 因 A2A 推进而逐步失效 | 高 | 中 | 短期保留 Wear RemoteInput 兜底（详见 §2.3.1）；中期切 A2A 协议或微信开放平台 API（详见 §2.3.2）；不可用时降级为"只朗读，不回复" |
| NotificationListenerService 无法 root 自动授权（S5） | 高 | 中 | appops 仅授权 POST_NOTIFICATIONS；通知使用权必须用户手动或 root 强制写 `settings put secure enabled_notification_listeners`（详见 §2.4）；推荐 APK 引导手动授权 |
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
| Q1 | 微信当前版本是否还暴露 Wear RemoteInput Action？2026-06 起 A2A 推进后是否仍可用？ | 假设短期仍可用，Phase 7 实施时实测；若失效则降级为"只朗读，不回复"并提前进入中期 A2A 调研 |
| Q2 | HyperOS 上 `cmd connectivity start-softap` 是否有效？ | 假设有，Phase 7 实施时实测 |
| Q3 | GLM-4V API 是否仍可用且价格可接受？ | 假设有，Phase 7 实施前复查 |
| Q4 | 来电朗读的时机如何把握（响铃几次后/接听后/挂断后）？ | 默认响铃 3 次后朗读 |
| Q5 | 是否需要为 NotificationReaderTool 单独的"勿扰时段"设置？ | 暂不做，用户可手动关闭 |
| Q6 | NotificationListenerService 授权是否走"APK 引导手动授权"路径，还是提供 root 强制 fallback？ | 默认走手动引导（详见 §2.4），root 强制仅作 power user escape hatch |
| Q7 | 微信 A2A 协议是否对第三方 APK 开放？若不开放，微信开放平台客服消息 API 是否可替代？ | 待 Phase 7+ 调研（详见 §2.3.2），不影响 Phase 7 MVP |
