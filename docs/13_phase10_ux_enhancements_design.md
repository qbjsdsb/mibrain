# Phase 10 设计：用户体验增强

> 本文档定义 Phase 10 的 6 个用户体验增强项设计。
> 阶段：设计草案（待 Phase 1-9 完成后实施）
> 依赖：Phase 1-2（APK 骨架 + 语音链路）+ Phase 7（手机控制）+ Phase 8（平台化）完成
> 与主设计关系：[00_design_overview.md](./00_design_overview.md) §11 路线图的扩展
> 与决策关系：[D1](../DECISIONS.md)（1.5B 默认影响儿童模式过滤能力）+ [D7](../DECISIONS.md)（JNI 影响模型热切换成本）+ [D21](../DECISIONS.md)（DE 路径影响多用户数据隔离）

> **2026-06-30 创建说明**：本文档在 Phase 0 设计冻结阶段创建，作为 Phase 10 路线占位与设计骨架。所有 G 项均与用户已确认的扩展功能清单一致，未引入新的决策项；涉及决策的部分（如 G2 儿童模式内容过滤强度、G5 i18n MVP 语言范围）标为开放问题留待 Phase 10 启动时讨论。

---

## 0. 设计目标

让 MiBrain 从"单人中文单设备"扩展为"全家多场景可用"的语音助手：

| G 项 | 做什么 | 主要受众 | 优先级建议 |
|---|---|---|---|
| G1: 多用户 | APK 内 profile 切换，独立聊天历史 + TTS 音色 + 唤醒词 | 共用手机的家人 | 高 |
| G2: 儿童模式 | profile 标 is_child，过滤 LLM 输出 + 限制工具调用 + 时长限制 | 有小孩的家庭 | 高 |
| G5: i18n | UI 多语言 + ASR/TTS/LLM 模型按语言切换 | 中英双语用户 | 中 |
| G6: a11y | TalkBack + 字体跟随 + 听障字幕 + 视障语音优先 | 视障/听障用户 | 中 |
| G9: 音频路由 | 蓝牙耳机/扬声器/听筒自动切换，TTS + ASR 路由跟随系统 | 蓝牙耳机用户 | 高 |
| G17: 全局暂停 | 双击电源键 / Widget / 通知按钮 → 任意状态原子切 PAUSED | 所有用户（紧急停止） | 高 |

**明确不做**（避免范围蔓延）：
- 完整 Android Multi-User（系统级多用户切换，复杂且 root 风险高，仅做 APK 内 profile）
- 儿童模式的家长远程监控（涉及云服务，与"完全离线"卖点冲突）
- 自定义 TTS 音色克隆（sherpa-onnx vits 不支持 zero-shot 克隆，留待后续模型升级）
- 手语翻译（多模态视频输出超出 MVP 范围）

---

## 1. G1: 多用户

### 1.1 现状与问题

默认设计是单用户：所有对话历史、TTS 配置、唤醒词模型共享。家庭共用手机时：
- 父亲用男声 TTS，母亲想用女声 → 每次切 TTS 模型需重新加载
- 父亲的对话历史被母亲看到 → 隐私问题
- 唤醒词都是 "hey jarvis"，但家人可能想用不同唤醒词

### 1.2 设计：APK 内 Profile 切换

不做 Android Multi-User，做 APK 内的 profile 切换。最多 3 个 profile（避免存储压力）。

**Profile 数据模型**（Room 表）：

```kotlin
@Entity(tableName = "profiles")
data class Profile(
    @PrimaryKey val id: String,              // UUID
    val name: String,                         // 显示名
    val avatarUri: String?,                   // 头像（可选）
    val isChild: Boolean = false,             // G2 儿童模式标记
    val ttsModelPath: String,                 // TTS 模型路径
    val wakewordModelPath: String,            // 唤醒词模型路径
    val asrModelPath: String? = null,         // ASR 模型路径（G5 i18n 时不同）
    val llmModelPath: String? = null,        // LLM 模型路径（默认走全局，按需覆盖）
    val createdAt: Long
)

@Entity(tableName = "conversations")
data class Conversation(
    @PrimaryKey val id: String,
    val profileId: String,                    // 关联 profile
    val text: String,
    val role: String,                         // user/assistant
    val timestamp: Long
)
```

### 1.3 切换流程

```
IDLE 状态
   ↓ 用户点"切换 Profile"
PROFILE_SWITCHING (CAS 原子转换)
   ↓ 1. 保存当前 profile 的对话上下文
   ↓ 2. 卸载当前 TTS 模型
   ↓ 3. 卸载当前唤醒词模型
   ↓ 4. 加载目标 profile 的 TTS + 唤醒词模型
   ↓ 5. 加载目标 profile 的对话上下文
   ↓ (失败) → 回滚到原 profile + 弹错误
IDLE 状态（新 profile）
```

**关键约束**：
- LLM 模型**不切换**（默认全局 1.5B），避免重新加载 ~1GB 模型耗时
- ASR 模型可切换（G5 i18n 时不同语言），但默认 zh-en bilingual 全 profile 共享
- 切换耗时预期：TTS 卸载+加载 ~2s + 唤醒词卸载+加载 ~1s = 总 ~3s

### 1.4 切换触发

- UI：设置页 → Profile 列表 → 点目标 profile
- 语音：未实现（多 profile 时无法用语音识别"我是谁"，留待未来）
- 通知栏：Foreground Service 通知加"切换 Profile"按钮（与 G17 共用通知栏）

### 1.5 与 DE 加密区的关系

所有 profile 的对话历史存 Room（自动落 DE 加密区 `/data/user_de/0/com.mibrain/databases/`），符合 [D21](../DECISIONS.md) Direct Boot 要求。模型文件共享 DE 区 `/data/user_de/0/com.mibrain/files/models/`，profile 仅引用路径不复制。

### 1.6 风险

| 风险 | 影响 | 缓解 |
|---|---|---|
| 切换中崩溃导致 profile 状态不一致 | 中 | PROFILE_SWITCHING 加超时（30s）+ 失败回滚 + 状态持久化到 Room |
| 3 个 profile 各自下载 TTS 模型导致存储压力 | 中 | TTS 模型共享，profile 仅记录路径；不同语言 TTS 才需新下载 |
| 对话历史泄露（root 可读 Room） | 低 | 设计文档明示 root 设备无强隔离；未来可加 SQLCipher 加密 |

---

## 2. G2: 儿童模式

### 2.1 现状与问题

默认设计无儿童过滤，LLM 可能输出不当内容；联网工具可能误触发（如搜索返回成人内容）；Phase 7 通知朗读会暴露家长隐私。

### 2.2 设计：Profile 级儿童模式

Profile 标记 `isChild=true` 时启用以下限制：

**限制项**：

| 项 | 默认 | 儿童模式 | 实现位置 |
|---|---|---|---|
| 联网工具 | 全开 | 仅 WeatherTool + TranslateTool | Phase 6 ToolRouter 检查 |
| Phase 7 工具 | 全开 | 仅 AppLauncherTool（白名单 app） | Phase 7 PhoneControlRouter 检查 |
| 通知朗读 | 全开 | 关闭 | NotificationReaderTool 禁用 |
| 截屏看屏 | 全开 | 关闭 | ScreenCaptureTool 禁用 |
| LLM 系统 prompt | 默认 | 加 "用儿童友好语言回答，不涉及暴力/成人/政治敏感内容" | LlamaEngine 调用时拼 prompt |
| 单次对话时长 | 无限制 | 5 分钟 | 状态机 LISTENING+THINKING+SPEAKING 总时长监控 |
| 每日累计时长 | 无限制 | 30 分钟（可配置） | ForegroundService 累计 + Room 记录 |
| 强制 TTS 音色 | 默认 | 轻快音色（可选 vits-zh-ll 子音色） | TTS 模型路径覆盖 |

### 2.3 双层过滤

**Pre-filter**（用户输入 → LLM 前）：
- 关键词黑名单（暴力/成人/政治）→ 命中时 LLM 直接回 "这个问题不太适合小朋友哦，换个话题吧"
- 正则识别"购买/付款/支付"→ 命中时拒绝联网工具调用

**Post-filter**（LLM 输出 → TTS 前）：
- 关键词黑名单扫描 LLM 输出
- 命中时替换为兜底文本 "我们聊点别的吧"
- 同时记录到 Room 表 `child_filter_hits` 供家长查看

### 2.4 时长限制实现

```kotlin
class ChildTimeLimiter(private val profileId: String, private val repo: ProfileRepository) {
    fun canStartSession(): Boolean {
        val todayUsed = repo.getTodayUsedMinutes(profileId)
        val limit = repo.getDailyLimit(profileId)  // 默认 30
        return todayUsed < limit
    }

    fun onSessionTick(state: ConversationState, durationMs: Long) {
        if (state in setOf(LISTENING, THINKING, SPEAKING)) {
            repo.addTodayUsed(profileId, durationMs)
            if (repo.getTodayUsedMinutes(profileId) >= repo.getDailyLimit(profileId)) {
                // 触发 COOLDOWN → IDLE，TTS 播 "今天玩够啦，明天再来吧"
            }
        }
    }
}
```

### 2.5 风险

| 风险 | 影响 | 缓解 |
|---|---|---|
| 1.5B 模型遵守儿童系统 prompt 能力弱 | 高 | 双层过滤兜底；~~不可靠时考虑儿童模式单独用 3B 模型~~（[D30](../DECISIONS.md) 已确认 3B 在 8GB 设备必 OOM，**8GB 设备不可切 3B**）；不可靠时改为强化系统 prompt + 关键词黑名单兜底，或 Phase 11+ 启用 Adreno GPU 加速后再评估 3B |
| 关键词黑名单误判 | 中 | 兜底文本温和，避免"拒绝感"；家长可在 Room 表查看过滤记录 |
| 时长限制绕过（改系统时间） | 低 | 不做防绕过，家长信任优先 |
| 儿童误切换 profile | 中 | 切换 profile 需 PIN（4 位数字，家长设置） |

---

## 3. G5: i18n（国际化）

### 3.1 现状与问题

默认中文 UI + 中文 ASR + 中文 TTS + Qwen2.5 LLM（中文好英文也行）。英文用户想用：
- UI 字符串英文
- ASR 识别英文
- TTS 英文音色
- LLM 英文能力强（Qwen2.5 中英都行，但纯英文场景 Llama 3.2 1B 更轻量）

### 3.2 设计：分层 i18n

| 层 | MVP 范围 | 可选扩展 |
|---|---|---|
| UI 字符串（strings.xml） | zh + en | 其他语言按需（用户贡献） |
| ASR 模型 | zh-en bilingual（默认） | en-only 模型可选 |
| TTS 模型 | vits-zh-ll（中文） + vits-vctk（英文） | 其他语言按需 |
| LLM 模型 | Qwen2.5-1.5B（中英都行，默认） | Llama 3.2 1B（英文场景可选） |
| 唤醒词 | hey_jarvis（默认） | 自训中文（[D15](../DECISIONS.md)） |

**MVP 范围明确为 zh + en 双语**，其他语言待社区贡献。

### 3.3 切换入口

- 设置 → 语言 → 选 zh / en / 跟随系统
- 切换时触发：
  - UI 立即重绘（Compose 自动 strings.xml 切换）
  - ASR/TTS 模型切换走 G1 PROFILE_SWITCHING 流程
  - LLM 模型默认不切换（Qwen2.5 1.5B 中英都行）；用户可选 "英文场景用 Llama 3.2 1B" 单独配置

### 3.4 模型体积压力

每个新语言的 ASR + TTS 增加 ~400MB：
- vits-vctk（英文 TTS）: ~150MB
- 英文 ASR 模型: ~250MB（可选，默认 zh-en bilingual 已覆盖）

**取舍**：默认配置仅含 zh-en bilingual ASR + vits-zh-ll TTS（~1.4GB）；用户切英文 UI 时提示"建议下载 vits-vctk 英文 TTS 以获得更好发音"。

### 3.5 风险

| 风险 | 影响 | 缓解 |
|---|---|---|
| 多语言模型累积存储压力 | 中 | 模型按需下载；不强制全装 |
| 中英混合场景 ASR 不稳 | 中 | streaming-zipformer-bilingual 已是中英双语模型，专门处理混合 |
| LLM 英文能力弱（1.5B） | 中 | 默认 Qwen2.5 1.5B 中英都行；纯英文场景可选 Llama 3.2 1B |

---

## 4. G6: a11y（无障碍）

### 4.1 现状与问题

默认设计假设用户能看屏幕 + 能听声音。视障用户用 TalkBack 听控件；听障用户看不到 ASR 文本也听不到 TTS。

### 4.2 设计：分场景 a11y

**视障**：
- 所有 Compose 控件加 `Modifier.semantics { contentDescription = "..." }`
- 状态变化语音播报：状态机 IDLE→LISTENING 时 TTS 播 "在听"（可关闭）
- 按键触觉反馈：HapticFeedback.composeVibrate()
- 语音优先：所有 UI 操作都有语音等价（如语音命令"暂停"等价于点暂停按钮）

**听障**：
- ASR 文本实时显示（已默认设计，确认 Phase 2 实现）
- TTS 文本同步显示（与 TTS 音频同步出现字幕）
- 视觉化状态：状态机变化时屏幕颜色变化（IDLE 灰 / LISTENING 蓝 / THINKING 黄 / SPEAKING 绿 / COOLDOWN 橙 / PAUSED 红）
- 振动反馈唤醒：唤醒词触发时震动 200ms

**通用**：
- 字体大小跟随系统 fontScale
- 颜色对比 WCAG AA（4.5:1）
- 高对比度模式可选（黑白主题）

### 4.3 实现要点

```kotlin
// 状态视觉化
val stateColor = when (state) {
    IDLE -> Color.Gray
    LISTENING -> Color.Blue
    THINKING -> Color.Yellow
    SPEAKING -> Color.Green
    COOLDOWN -> Color Orange
    PAUSED -> Color.Red
}
Box(modifier = Modifier.background(stateColor)) { ... }

// TTS 字幕同步
class TTSSubtitleBinder {
    fun onTTSChunk(text: String) {
        // 实时追加到 UI 字幕区
    }
    fun onTTSEnd() {
        // 字幕停留 3s 后清除
    }
}
```

### 4.4 风险

| 风险 | 影响 | 缓解 |
|---|---|---|
| Compose a11y 工程量大 | 中 | Phase 10 专项迭代，先做核心控件后做次要 |
| TTS 字幕同步精度 | 中 | sherpa-onnx TTS 有 chunk 回调，按 chunk 显示而非按字 |
| 与 TTS 语音反馈重复（本来就有语音） | 低 | 视障模式默认开语音，听障模式默认关语音开字幕 |

---

## 5. G9: 音频路由

### 5.1 现状与问题

默认 AudioRecord 用麦克风 + AudioTrack 用扬声器。用户接蓝牙耳机后期望：
- TTS 从蓝牙耳机出
- ASR 用蓝牙耳机麦克风
- 切换时不中断当前对话

### 5.2 设计：跟随系统音频路由

**原则**：MiBrain 不自定路由策略，跟随系统当前输出/输入设备。

**TTS 输出**：
- AudioTrack 用 `AudioAttributes` 标 `USAGE_ASSISTANT` + `CONTENT_TYPE_SPEECH`
- 系统自动路由到当前输出设备（蓝牙 / 扬声器 / 听筒）
- 切换中：监听 `AudioDeviceCallback`，设备变化时不中断 AudioTrack（系统自动切）

**ASR 输入**：
- AudioRecord 用 `MediaRecorder.AudioSource.VOICE_RECOGNITION`
- 系统自动选当前输入设备（蓝牙麦克风 / 内置麦克风）
- 切换中：监听设备变化，重启 AudioRecord（不可避免，~500ms 静音）

### 5.3 蓝牙采样率问题

蓝牙耳机麦克风默认采样率 48kHz / 16kHz，但部分蓝牙耳机只支持 8kHz（HFP 模式）。
- AudioRecord 创建时请求 16kHz，系统自动协商
- 若协商到非 16kHz，APK 内做重采样（ sherpa-onnx 要 16kHz）

### 5.4 状态机扩展

路由切换时：
- IDLE 状态：忽略，下次 LISTENING 自动用新设备
- LISTENING 状态：重启 AudioRecord（500ms 静音期，TTS 播 "正在切换音频设备"）
- SPEAKING 状态：不中断（系统自动切输出设备）
- THINKING 状态：无影响

### 5.5 风险

| 风险 | 影响 | 缓解 |
|---|---|---|
| Phantom Mic 在蓝牙下行为未知 | 高 | Phase 4 真机验证蓝牙场景；失败时 fallback 到内置麦克风 + 提示用户 |
| 蓝牙麦克风延迟大 | 中 | VAD 阈值调宽，避免误判结束 |
| 重采样引入 ASR 精度损失 | 低 | Android 内置重采样质量够用 |
| 切换中 LISTENING 静音 500ms | 低 | TTS 播报 + 用户预期管理 |

---

## 6. G17: 全局暂停

### 6.1 现状与问题

用户想立刻停止 MiBrain（被打断、紧急情况、临时不想用）时：
- 关闭 Foreground Service 太慢（要去设置）
- 退出 APK 不彻底（Foreground Service 仍在跑）
- 锁屏不暂停（设计目标是锁屏也能唤醒）

需要"任意状态 → 立即暂停"的快捷方式。

### 6.2 设计：多入口暂停

| 入口 | 实现 | 优先级 |
|---|---|---|
| 双击电源键 | 监听 KeyEvent.KEYCODE_POWER + LSPosed 注入或 root 注入 | 主推（手最快） |
| 桌面 Widget | 1x1 Widget，单点即暂停（与 Phase 8 Cap 3 共用） | 必备（兜底） |
| 通知按钮 | Foreground Service 通知加"暂停"action | 必备（兜底） |
| 语音命令 | 说"暂停" → ASR 识别 → 状态机切 PAUSED | 增强（不可靠，留作 fallback） |

### 6.3 状态机扩展

新增 `PAUSED` 状态：

```
任意状态（除 PROFILE_SWITCHING）
   ↓ 双击电源键 / 点 Widget / 点通知按钮
PAUSED (CAS 原子转换，无条件)
   ↓ 双击电源键 / 点 Widget / 点通知按钮 / 5 分钟超时
IDLE
```

**PAUSED 状态行为**：
- 停止 TTS（立即静音）
- 停止 ASR（释放 AudioRecord）
- 停止唤醒词检测（释放 KWS）
- 释放 WakeLock
- **不卸载模型**（5 分钟内恢复时不需重新加载）
- 5 分钟超时 → 自动回 IDLE 并重新 acquire WakeLock
- 通知栏显示"已暂停 - 点击恢复"

### 6.4 双击电源键实现

**方案 A（推荐）**：LSPosed 注入 PhoneWindowManager
- hook `interceptKeyBeforeQueueing` 监听 KEYCODE_POWER
- 检测 300ms 内双击 → 广播 `com.mibrain.PAUSE_TOGGLE`
- 需要新 LSPosed 模块或扩展现有 Phantom Mic scope

**方案 B（fallback）**：root 注入 + AccessibilityService
- AccessibilityService 监听系统按键事件（部分 ROM 受限）
- 不可靠，作为 fallback

**方案 C（最 fallback）**：仅 Widget + 通知按钮
- 不依赖 LSPosed / root 注入
- 工程量最小，但用户需点屏幕

**MVP 推荐**：方案 C（Widget + 通知按钮）+ 方案 A 作为 Phase 10 增强项。

### 6.5 风险

| 风险 | 影响 | 缓解 |
|---|---|---|
| 双击电源键在 MIUI 上被系统拦截 | 高 | 方案 C 兜底；方案 A 需 LSPosed 注入工程量 |
| PAUSED 状态忘记恢复 | 低 | 5 分钟超时自动回 IDLE + 通知栏提示 |
| 模型常驻 5 分钟内存压力 | 中 | PAUSED 5 分钟超时后卸载模型回 IDLE；可配置超时 |
| Widget 在 HyperOS 3 上被限制 | 中 | 通知按钮作为 Widget 的 fallback |

---

## 7. 状态机扩展汇总

各 G 项对状态机的影响：

| G 项 | 新增状态 | 转换 | CAS 实现 |
|---|---|---|---|
| G1 多用户 | PROFILE_SWITCHING | IDLE → PROFILE_SWITCHING → IDLE | AtomicReference.compareAndSet |
| G2 儿童模式 | 无 | 时长超限时 THINKING/SPEAKING → COOLDOWN → IDLE | 复用现有 COOLDOWN |
| G5 i18n | 无 | 模型切换走 G1 PROFILE_SWITCHING | 复用 G1 |
| G6 a11y | 无 | 无新状态，仅 UI 反馈 | - |
| G9 音频路由 | 无 | LISTENING 重启 AudioRecord（500ms 静音期） | 复用现有 LISTENING |
| G17 全局暂停 | PAUSED | 任意状态 → PAUSED → IDLE | AtomicReference.compareAndSet |

**完整状态机**：

```
IDLE
  ↓ KWS 命中
LISTENING
  ↓ VAD 结束
THINKING
  ↓ LLM 完成
SPEAKING
  ↓ TTS 完成
COOLDOWN
  ↓ 1s
IDLE

# G1 扩展
IDLE
  ↓ 用户切换 profile
PROFILE_SWITCHING
  ↓ 完成/失败
IDLE (新 profile)

# G17 扩展
任意状态（除 PROFILE_SWITCHING）
  ↓ 暂停触发
PAUSED
  ↓ 恢复触发/5min 超时
IDLE
```

---

## 8. 数据模型扩展

汇总各 G 项对 Room 表的影响：

```kotlin
// G1: profile 表
@Entity(tableName = "profiles")
data class Profile(...)

// G1: 对话历史关联 profile
@Entity(tableName = "conversations")
data class Conversation(... profileId ...)

// G2: 儿童过滤记录
@Entity(tableName = "child_filter_hits")
data class ChildFilterHit(
    @PrimaryKey val id: String,
    val profileId: String,
    val originalText: String,
    val matchedKeyword: String,
    val timestamp: Long
)

// G2: 儿童每日时长
@Entity(tableName = "child_daily_usage")
data class ChildDailyUsage(
    @PrimaryKey val profileId: String,
    val date: String,                  // YYYY-MM-DD
    val usedMs: Long
)

// G17: 暂停超时配置
@Entity(tableName = "settings")
data class Setting(
    @PrimaryKey val key: String,
    val value: String
)
// key: "pause_timeout_ms" / "child_daily_limit_min" / "i18n_lang" 等
```

---

## 9. 风险表

汇总各 G 项风险（详见各 §）：

| G 项 | 主要风险 | 等级 | 缓解 |
|---|---|---|---|
| G1 | profile 切换中崩溃导致状态不一致 | 中 | PROFILE_SWITCHING 加超时 + 失败回滚 |
| G2 | 1.5B 模型遵守儿童系统 prompt 能力弱 | 高 | 双层过滤兜底；**3B 在 8GB 设备不可用（[D30](../DECISIONS.md)），不可切 3B**，只能强化 prompt + 黑名单 |
| G5 | 多语言模型存储压力 | 中 | 按需下载 |
| G6 | Compose a11y 工程量大 | 中 | Phase 10 专项迭代 |
| G9 | Phantom Mic 在蓝牙下行为未知 | 高 | Phase 4 真机验证 + fallback |
| G17 | 双击电源键被系统拦截 | 高 | MVP 仅 Widget + 通知按钮 |

**整体风险**：G2 和 G9 风险最高，需 Phase 4 真机验证后才能确认设计可行。

---

## 10. 与其他 Phase 的关系

| Phase | 关系 |
|---|---|
| Phase 1 | G1 profile 切换走 PROFILE_SWITCHING 状态机，依赖 Phase 1 状态机基础 |
| Phase 2 | G5 i18n ASR/TTS 模型切换依赖 Phase 2 sherpa-onnx 集成 |
| Phase 6 | G2 儿童模式限制联网工具，依赖 Phase 6 ToolRouter |
| Phase 7 | G2 儿童模式限制手机控制工具，依赖 Phase 7 PhoneControlRouter |
| Phase 8 Cap 3 | G17 暂停 Widget 与 Phase 8 Cap 3 桌面 Widget 共用 |
| Phase 9 | 无直接关系 |

---

## 11. 开放问题

| 问题 | 待讨论时机 |
|---|---|
| G1 profile 最大数：3 还是 5？ | Phase 10 启动时 |
| G2 儿童模式是否单独用 3B 模型（牺牲内存换过滤遵守度）？ | **已答：3B 在 8GB 设备必 OOM（[D30](../DECISIONS.md)），不可切 3B**；改为强化 prompt + 关键词黑名单兜底 |
| G2 关键词黑名单由谁维护？硬编码 vs 设置页可编辑 | Phase 10 启动时 |
| G5 i18n 是否要支持日韩等语言？还是仅中英双语 | Phase 10 启动时（建议仅中英） |
| G6 a11y 是否要支持盲文输出？ | 暂不做，未来考虑 |
| G9 是否要支持 USB 外接麦克风？ | 暂不做，跟随系统自动协商 |
| G17 双击电源键是否值得做 LSPosed 注入工程量？ | Phase 10 启动时评估，MVP 可仅 Widget + 通知 |

---

## 12. 实施优先级建议

按用户痛点和工程量排序：

| 优先级 | G 项 | 理由 |
|---|---|---|
| P0（必做） | G17 全局暂停 | 所有用户都需要紧急停止；MVP 仅 Widget + 通知按钮，工程量小 |
| P0（必做） | G9 音频路由 | 蓝牙耳机用户基础需求；跟随系统路由工程量小 |
| P1（应做） | G1 多用户 | 家庭共用场景；profile 切换工程量中等 |
| P1（应做） | G2 儿童模式 | 有小孩的家庭必需；双层过滤工程量中等 |
| P2（可选） | G6 a11y | 视障/听障用户少数；Compose a11y 工程量大 |
| P2（可选） | G5 i18n | 中英双语已是默认；完整 i18n 需多语言模型下载支持 |

**MVP Phase 10 范围建议**：G17（Widget + 通知按钮）+ G9 + G1 + G2，G6 和 G5 留作 Phase 11。

---

## 13. 决策清单

本文档引入的待确认决策（待 Phase 10 启动时讨论，未立即加入 [DECISIONS.md](../DECISIONS.md)）：

| 决策 | 候选 | 待确认 |
|---|---|---|
| profile 最大数 | 3 / 5 | Phase 10 启动 |
| 儿童模式 LLM 模型 | **仅 1.5B（3B 在 8GB 必 OOM，[D30](../DECISIONS.md)）** | 已确认 |
| i18n MVP 语言范围 | 仅中英 / 中英日韩 | Phase 10 启动 |
| G17 双击电源键 | LSPosed 注入 / 仅 Widget | Phase 10 启动 |
| 关键词黑名单维护方式 | 硬编码 / 设置页可编辑 | Phase 10 启动 |
