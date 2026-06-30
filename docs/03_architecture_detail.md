# 详细架构与时序图

> 本文档补充 `00_design_overview.md` 中未写清的运行时细节：
> JNI 推理后端、模型存储路径、回环防护、状态机 CAS、启动顺序。
>
> **2026-06-30 重大修订**：
> - 推理后端从 llama-server HTTP 切回 JNI（[D7](../DECISIONS.md)），详见 [X7 废弃](../DECISIONS.md)
> - 模型路径改 DE + Direct Boot（[D21](../DECISIONS.md)），解决 S1 + S2
> - 默认模型从 3B 改 1.5B（[D1](../DECISIONS.md)），解决 S4 内存预算
> - 唤醒词改 sherpa-onnx KWS（[D23](../DECISIONS.md)），弃用 openWakeWord
> - ASR/TTS 模型换 Apache 2.0 许可（[D22](../DECISIONS.md)），解决 S7

---

## 1. 进程拓扑（运行时）

```
┌─────────────────────────────────────────────────────────────────────┐
│ Android 系统进程（init 起）                                            │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ MiBrain APK (uid=10xxx, 非 root)                                │ │
│  │   - ForegroundService (microphone type)                         │ │
│  │   - AudioRecord 持续录音                                         │ │
│  │   - LlamaEngine.kt（JNI 调用 libllama.so）                     │ │
│  │   - sherpa-onnx ASR/TTS/VAD/KWS in-process（单一 AAR）          │ │
│  │   - 模型路径：/data/user_de/0/com.mibrain/files/models/        │ │
│  │     （DE 加密区，Direct Boot 下可读）                             │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ LSPosed 注入（root 后启动）                                       │ │
│  │   - Phantom Mic hook AudioRecord.cpp native 层                  │ │
│  │   - 让 APK 在锁屏/后台仍能持续拿到麦克风数据                       │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

**架构变化要点**（vs 原 HTTP 设计）：
- **不再有 root 启动的 llama-server 子进程**，推理在 APK 内通过 JNI 完成
- **不再需要 KSU service.sh 启动推理服务**（KSU 仅用于 Phantom Mic 等系统级配置）
- **不再有 127.0.0.1:8080 HTTP 端口**（除非 Phase 8 Cap 1 启用本地 API 暴露）
- APK 体积增加 ~30MB（含 arm64-v8a 的 libllama.so + libggml.so + sherpa-onnx .so，实测数据来自日本 Pasona 部署实录：所有 .so 文件合计约 30MB）

### 关键文件路径

| 资源 | 路径 | 说明 |
|---|---|---|
| 模型目录 | `/data/user_de/0/com.mibrain/files/models/` | DE 加密区，Direct Boot 下可读 |
| LLM 模型 | `${model_dir}/qwen2.5-1.5b-instruct-q4_k_m.gguf` | 默认 1.5B，~1GB |
| ASR 模型 | `${model_dir}/sherpa-streaming-zipformer-bilingual-zh-en/` | Apache 2.0 |
| TTS 模型 | `${model_dir}/sherpa-vits-zh-ll/` | 社区贡献，许可未明确声明（替代 aishell3，[D22](../DECISIONS.md)） |
| VAD 模型 | `${model_dir}/silero_vad.onnx` | sherpa-onnx 自带 |
| KWS 模型 | `${model_dir}/hey_jarvis.onnx` | sherpa-onnx KWS 训练产出 |
| 配置文件 | `/data/user_de/0/com.mibrain/files/config.json` | 用户可改的设置 |
| 日志目录 | `/data/user_de/0/com.mibrain/files/logs/` | 滚动，单文件 1MB，保留 3 份 |

---

## 2. APK 启动流程

### 2.1 Direct Boot 支持

为支持锁屏唤醒（Phase 4 验收标准），APK 必须在 `android:directBootAware="true"` 声明，让 ForegroundService 在用户首次解锁前能启动。

```xml
<!-- AndroidManifest.xml -->
<application
    android:directBootAware="true"
    ...>
    <service
        android:name=".service.MiBrainForegroundService"
        android:foregroundServiceType="microphone"
        android:directBootAware="true"
        android:exported="false" />
</application>
```

### 2.2 启动顺序

```
T=0     开机
T=10s   KSU post-fs-data.sh 配置 appops（RECORD_AUDIO allow 等）
T=12s   系统启动完成
T=20s   LSPosed 注入 Phantom Mic hook
T=30s   MiBrain ForegroundService 启动（Direct Boot 模式下，无需用户解锁）
T=32s   ForegroundService 检测 DE 区模型是否就位
        ├── 已就位 → JNI 加载 LLM + sherpa-onnx 初始化 ASR/TTS/VAD/KWS
        └── 未就位 → 进入"等待模型下载"状态，TTS 提示用户首次需联网下载
T=35s   模型加载完成（1.5B，~3-5s）
T=36s   state = IDLE，激活唤醒词检测
T=60s   用户首次解锁（如已设置 PIN），CE 区可访问
```

### 2.3 模型下载流程（首次启动）

```
APK 首次启动
  ↓
检测 DE 区 ${model_dir}/qwen2.5-1.5b-q4_k_m.gguf
  ↓ 不存在
显示"需要下载模型 (~1GB)，请连接 Wi-Fi"
  ↓ 用户确认
启动下载服务（带进度 UI + 断点续传 + SHA256 校验）
  ↓ 多源切换：HuggingFace 主 → 阿里 OSS 镜像兜底
  ↓ 下载完成
SHA256 校验
  ├── 通过 → JNI 加载模型 → 进入 IDLE 状态
  └── 失败 → 删除文件 → 提示重试
```

### 2.4 失败处理

| 故障 | 检测 | 处理 |
|---|---|---|
| 模型文件不存在 | ForegroundService 启动时检测 | 进入"等待模型下载"状态，引导用户 |
| 模型加载失败（OOM） | JNI native 抛 OOM 异常 | 提示"内存不足"，建议关闭其他 app |
| 模型 SHA256 不匹配 | 下载后校验 | 删除文件，提示重新下载 |
| LSPosed 框架未启用 | APK 启动时检测 `/data/adb/lspd` 是否存在 | 显示"请启用 LSPosed 框架" |
| Phantom Mic 未启用 | APK 启动时检测 LSPosed scope | 显示"请启用 Phantom Mic 范围" |
| ZygiskNext 未启用 | APK 启动时检测 `/data/adb/modules/zygisksu` | 显示"请启用 ZygiskNext" |
| APK 被 MIUI 杀 | ForegroundService 死 | 通知 + 自启动白名单引导 |
| onTrimMemory(CRITICAL) | 系统内存压力 | JNI 卸载模型，下次唤醒时重新加载（接受 3-5s 加载延迟） |

---

## 3. JNI 推理接口（LlamaEngine.kt）

> **命名说明**：本项目自命名 `LlamaEngine.kt`，与 ToolNeuron 实际文件名 `InferenceService.kt` 区分；实现思路参考 ToolNeuron 的 `InferenceService.kt`（位于 `service/inference/` 目录）与 llama.android 官方模块的 `InferenceEngine`，并非 fork ToolNeuron 的同名文件。

### 3.1 接口签名（草案）

```kotlin
class LlamaEngine(private val modelPath: String, private val config: LlamaConfig) {
    // 加载模型，返回是否成功
    external fun loadModel(): Boolean

    // 一次性对话（Phase 1 MVP）
    external fun complete(prompt: String, maxTokens: Int, temperature: Float): String

    // 流式对话（Phase 2 起），通过 callback 回调每个 token
    external fun streamComplete(
        prompt: String,
        maxTokens: Int,
        temperature: Float,
        callback: (token: String) -> Unit
    ): String

    // 卸载模型释放内存（onTrimMemory 时调用）
    external fun unloadModel()

    // 获取模型版本与能力（用于 [M3](#m3-apk--ksu-模块之间没有版本协商机制) 版本协商）
    external fun getVersion(): LlamaVersion
}

data class LlamaVersion(
    val llamaCppBuild: String,    // "b9844"
    val capabilities: List<String>, // ["chat", "tools", "stream"]
    val modelLoaded: Boolean
)
```

### 3.2 流式输出切句策略

APK 按句号/问号/感叹号/换行切句，每完整一句立即送给 sherpa-onnx TTS，不等流结束。与原 HTTP SSE 方案行为一致，只是数据通过 JNI callback 而非 SSE。

### 3.3 版本协商（解决 M3）

APK 启动时调 `LlamaEngine.getVersion()`，校验：
- `llamaCppBuild` 是否 >= 期望版本（避免老版本不支持新 API）
- `capabilities` 是否含 `tools`（Phase 6 function calling 依赖）
- 不通过时显示"内置 llama.cpp 版本过旧，请升级 APK"

### 3.4 keep-alive 卸载策略

5 分钟无活动 → 调 `unloadModel()` 释放模型内存。下次唤醒时重新加载（接受 3-5s 延迟）。比 HTTP 方案的 `/unload` 端点更直接。

---

## 4. 回环防护协议（Phase 2）

### 问题
TTS 播放声音 → 扬声器 → 麦克风拾起 → ASR 识别 → 当作新命令处理。

### 解决：状态机 + CAS 原子转换

```
状态枚举:
  IDLE              等待唤醒
  LISTENING         正在录音 ASR
  THINKING          LLM 推理中
  SPEAKING          TTS 播放中
  COOLDOWN          播放完 500ms 冷却（防回声尾巴）
  TOOL_RUNNING      联网/手机控制工具执行中（Phase 6/7 新增）
  IMAGE_CAPTURING   等待用户拍照（Phase 9 新增）

状态转移（统一通过 CAS 原子操作，解决 M4 并发抢占）:
  IDLE --(唤醒词检出)--> LISTENING
  IDLE --(通知到达，Phase 7)--> SPEAKING
  IDLE --(用户主动触发工具)--> TOOL_RUNNING
  LISTENING --(VAD 静音 1s)--> THINKING
  THINKING --(首 token 出)--> SPEAKING
  THINKING --(LLM 决定调工具)--> TOOL_RUNNING
  TOOL_RUNNING --(工具完成)--> THINKING
  TOOL_RUNNING --(需拍照，Phase 9)--> IMAGE_CAPTURING
  IMAGE_CAPTURING --(用户拍完)--> TOOL_RUNNING
  SPEAKING --(播放完毕 + 500ms)--> COOLDOWN
  COOLDOWN --(立即)--> IDLE
  (SPEAKING，多轮模式 Phase 9) --> COOLDOWN --> LISTENING  # 多轮对话接续听

约束:
  - 只在 IDLE 状态下激活唤醒词检测
  - 只在 LISTENING 状态下喂 ASR 数据
  - SPEAKING + COOLDOWN 期间丢弃所有麦克风数据
  - TOOL_RUNNING 期间 TTS 播报"正在查询..."
  - IMAGE_CAPTURING 60s 超时 → 强制回 IDLE
  - 每个状态有最大停留时间（如 THINKING > 30s 强制回 IDLE + TTS "超时"）
```

### 协议层实现（解决 M4 并发抢占）

```kotlin
// 用 compareAndSet 而非 AtomicReference.set，保证 check + transition 原子
private val state = AtomicReference(ConversationState.IDLE)

fun transitionTo(expected: ConversationState, new: ConversationState): Boolean {
    return state.compareAndSet(expected, new)
}

// 唤醒路径
fun onWakeWordDetected() {
    if (transitionTo(ConversationState.IDLE, ConversationState.LISTENING)) {
        // 成功转 LISTENING，启动 ASR
    } else {
        // 当前非 IDLE（可能在 SPEAKING / TOOL_RUNNING），丢弃此唤醒
    }
}

// 通知路径（Phase 7）
fun onNotificationArrived() {
    if (transitionTo(ConversationState.IDLE, ConversationState.SPEAKING)) {
        // 成功转 SPEAKING，朗读通知
    } else {
        // 当前非 IDLE，入队等待
        notificationQueue.offer(latestNotification)
    }
}
```

### WakeLock（解决 M12）

```
IDLE → LISTENING：acquire PARTIAL_WAKE_LOCK
COOLDOWN → IDLE：release WAKE_LOCK
```

避免对话中设备休眠导致 AudioRecord.read() 返回 0 或卡住。

### AudioFocus 协作

TTS 播放前请求 AudioFocus（TRANSIENT 类型），让其他 App 媒体暂停。Phase 7 来电场景下 AudioFocus 是必须——否则铃声 + TTS 同时播放听不清。

---

## 5. 关键时序图（一次完整对话）

```
用户              APK                    LlamaEngine(JNI)    sherpa-onnx      AudioTrack
 │                  │                       │                    │                │
 │ "hey jarvis"    │                       │                    │                │
 ├─────────────────►│                       │                    │                │
 │                  │ 唤醒检出 80ms          │                    │                │
 │                  │ state=LISTENING       │                    │                │
 │                  │ acquire WakeLock       │                    │                │
 │                  │ ───────────────────────────────────────────►│ ASR            │
 │ "明天天气"      │                       │                    │                │
 ├─────────────────►│                       │                    │                │
 │                  │ ◄───────────────────────────────────────────┤ "明天天气"     │
 │                  │ state=THINKING        │                    │                │
 │                  │ ──streamComplete────►│                    │                │
 │                  │                       │ llama_decode      │                │
 │                  │ ◄──callback token────┤ "明天"             │                │
 │                  │ ◄──callback token────┤ "明天晴"           │                │
 │                  │ 切句检测"。"           │                    │                │
 │                  │ state=SPEAKING        │                    │                │
 │                  │ ──────────────────────────────────────────►│ TTS            │
 │                  │ ◄──────────────────────────────────────────┤ audio chunk    │
 │                  │ ───────────────────────────────────────────────────────────►│ 播放
 │ 听到"明天晴"     │                       │                    │                │
 │                  │ 播放完毕              │                    │                │
 │                  │ state=COOLDOWN (500ms)│                   │                │
 │                  │ state=IDLE            │                    │                │
 │                  │ release WakeLock      │                    │                │
 │                  │ 重新激活唤醒检测       │                    │                │
```

总延迟：
- 唤醒检出: 80ms
- ASR 转写: 1.5-3s（5s 录音）
- LLM 首 token: ~1s
- 等首句完成（按 ~10 token）: ~2s
- TTS 合成首句: 0.3s
- **首句总延迟: ~5-7s**

---

## 6. 内存预算（2026-06-30 第三轮严格重算，修正算术错误 E10）

> ⚠️ **2026-06-30 第三轮修订**：原表常驻合计 6.32GB、峰值合计 6.77GB、headroom 1.23GB **均有算术错误**（约低估 0.7-0.8GB）。本轮严格重算后：
> - 1.5B 常驻 **6.64GB** / 峰值（叠加最坏）**7.89GB** / 串行峰值约 **7.5GB** / headroom 仅 **0.11-0.5GB**（**紧张，Stage 5 真机硬关卡**）
> - 3B 常驻 **7.54GB** / 峰值 **8.59GB**（**8GB 设备必 OOM，3B 选项不可用**）
> - 同时修正 libllama.so + libggml.so 体积：~50MB → ~30MB（日本 Pasona 实测，[W1](./14_feasibility_recheck_and_plan.md)）
> - 同时修正 sherpa-onnx .so 体积：~80MB → ~19MB（只打 arm64-v8a 实测：libsherpa-onnx-jni.so 3.7M + libonnxruntime.so 15M）
> - 同时修正 Qwen2.5-1.5B 内存：原 0.9/1.2GB 偏低，实际 1.5B Q4_K_M 权重 0.9GB + KV cache 2048 ctx + 推理缓冲区，保守估常驻 1.3GB / 峰值 1.7GB

8GB 总内存，默认 1.5B 模型（按 E10 修订后严格算法，所有"峰值"列累加）：

| 组件 | 常驻内存 | 峰值内存 | 备注 |
|---|---|---|---|
| HyperOS 3 + 系统服务 | 5.0GB | 5.5GB | 不可压缩（实测可能 5.5-6GB）；HyperOS 3 已对 K50U 推送（[F2](./14_feasibility_recheck_and_plan.md)），仍是 Android 15 |
| LSPosed Vector runtime | 100MB | 100MB | Vector v2.0.3-7716（2026-05-20 发布，[F1](./14_feasibility_recheck_and_plan.md)） |
| Phantom Mic (LSPosed) | 30MB | 30MB | v2.0（2024-07，待复查上游活跃度，[D14](../DECISIONS.md)） |
| MiBrain APK 基础 | 100MB | 100MB | Compose + AndroidX |
| libllama.so + libggml.so | 30MB | 30MB | JNI 库本身（日本 Pasona 实测 ~30MB，[W1](./14_feasibility_recheck_and_plan.md)） |
| sherpa-onnx .so + onnxruntime | 19MB | 19MB | 一个 AAR 覆盖 ASR/TTS/VAD/KWS，仅打 arm64-v8a 实测（libsherpa-onnx-jni.so 3.7M + libonnxruntime.so 15M） |
| **Qwen2.5-1.5B Q4_K_M** | **1.3GB** | **1.7GB** | **权重 0.9GB + KV cache 2048 ctx + 推理缓冲区**（保守估算，Phase 4 真机实测修正） |
| sherpa-onnx VAD 常驻 | 30MB | 30MB | silero_vad，必须常驻 |
| sherpa-onnx KWS 常驻 | 30MB | 30MB | 必须常驻（否则无法唤醒） |
| sherpa-onnx ASR 推理 | - | 200MB | 仅 LISTENING 期间 |
| sherpa-onnx TTS 推理 | - | 150MB | 仅 SPEAKING 期间 |
| **常驻合计（1.5B 默认）** | **6.64GB** | - | 留 1.36GB headroom |
| **峰值合计（1.5B 默认，叠加最坏）** | - | **7.89GB** | 所有组件峰值累加（保守上限） |
| **峰值合计（1.5B 默认，串行实际）** | - | **~7.5GB** | ASR→LLM→TTS 串行，LLM 阶段峰值最高（常驻 + LLM 推理增量） |

**结论（默认 1.5B）**：
- 串行峰值约 7.5GB，留 **0.5GB** headroom，**紧张但可接受**
- 最坏叠加峰值 7.89GB，留 **0.11GB** headroom，**极度紧张，Stage 5 真机 24h 测试必做**
- Stage 5 准出条件修正为：**串行峰值 < 7.6GB + 24h 无 OOM kill**（原 7GB 已无参考意义）

**对比 3B 模型（质量优先模式）**：

| 组件 | 3B 常驻 | 3B 峰值 |
|---|---|---|
| Qwen2.5-3B Q4_K_M | 1.8GB | 2.2GB |（权重 1.8GB + KV cache + 推理缓冲区） |
| **常驻合计（3B）** | **7.54GB** | - | 已逼近 8GB 上限 |
| **峰值合计（3B，叠加最坏）** | - | **8.59GB** | **> 8GB，必 OOM** |
| **峰值合计（3B，串行实际）** | - | **~8.2GB** | **> 8GB，必 OOM** |

⚠️ **3B 模型在 8GB 设备上不可行**（[D1](../DECISIONS.md) 备选条件修订）：常驻已 7.54GB，叠加峰值 8.59GB > 8GB RAM，**会触发 lmkd 杀进程**。
- 原 [D1](../DECISIONS.md) "3B 作为质量优先可选项" 的前提（峰值 7.67GB）已被本轮重算推翻
- **修订建议**：3B 选项从 D1 "质量优先可选项" 降级为 "8GB 设备不可用，仅 12GB+ 设备可启用"，或在 Phase 11 启用 Adreno GPU 加速后才考虑
- **关联**：[D25](../DECISIONS.md) Qwen2.5-1.5B function calling 限制——3B 既不可用，function calling 只能靠 1.5B + 32-token CoT prompt，复杂工具调用可靠性进一步受限

**VAD + KWS 常驻的必要性**：
- VAD 必须常驻：否则没法检测用户何时开始说话
- KWS 必须常驻：否则没法被"hey jarvis"唤醒
- 两者之和 60MB，加上 onnxruntime 80MB = 140MB，**无法通过"串行执行"释放**

如果系统压力上来，APK 监听 `onTrimMemory(TRIM_MEMORY_RUNNING_CRITICAL)` 主动调 `unloadModel()` 卸载 LLM 模型，下次唤醒时重新加载（接受 3-5s 加载延迟）。VAD + KWS 不卸载。

---

## 7. 协议错误码（仅 Phase 8 Cap 1 启用本地 API 暴露时适用）

JNI 路径下，APK 内调用无 HTTP 协议。但 Phase 8 Cap 1 启用本地 API 暴露后，外部 app 通过 HTTP 调用 MiBrain：

| HTTP Code | 含义 | 调用方处理 |
|---|---|---|
| 200 | 正常 | 解析响应 |
| 400 | 请求格式错 | 显示"内部错误"，记日志 |
| 500 | LLM 推理内部错 | 重试一次，仍失败提示用户重启 |
| 503 | 模型未加载 | 显示"正在加载模型..." |
| 429 | 限流（Phase 8 Cap 1） | 退避重试 |
| timeout | 服务无响应 | 显示"服务无响应" |

---

## 8. 配置文件 schema（2026-06-30 修订）

`/data/user_de/0/com.mibrain/files/config.json`：

```json
{
  "schema_version": 1,
  "model": {
    "path": "qwen2.5-1.5b-instruct-q4_k_m.gguf",
    "ctx_size": 2048,
    "threads": 4,
    "keep_alive_minutes": 5,
    "gpu_layers": 0
  },
  "asr": {
    "model_dir": "sherpa-streaming-zipformer-bilingual-zh-en",
    "vad_threshold": 0.5,
    "silence_duration_ms": 1000
  },
  "tts": {
    "model_dir": "sherpa-vits-zh-ll",
    "speaker_id": 0,
    "speed": 1.0
  },
  "wakeword": {
    "model": "hey_jarvis.onnx",
    "threshold": 0.5,
    "cooldown_ms": 5000
  },
  "system_prompt": "你是 MiBrain，一个本地运行的语音助手。简洁回答。",
  "network": {
    "enabled": false,
    "tools": {
      "weather": true,
      "translate": true,
      "search": false,
      "news": true
    },
    "api_keys": {
      "qweather": "",
      "bing": ""
    }
  }
}
```

**变更说明**：
- 路径全部改为相对路径（基于 `${model_dir}`），不再用绝对 `/sdcard/...`
- 新增 `schema_version` 字段（解决 O9 schema 演进）
- `model.path` 改 1.5B 默认
- ASR 改 Apache 2.0 许可的 streaming-zipformer-bilingual
- TTS 改 vits-zh-ll（社区贡献，许可未明确声明，发布前需替换为 matcha-icefall-zh-baker）
- 唤醒词模型改 sherpa-onnx KWS
- 新增 `network` 配置块（Phase 6 联网开关）

APK 启动时读这个文件，UI 提供修改界面。schema_version 不匹配时提示升级。

---

## 9. 待补充事项

以下细节在 Phase 1-2 工程实现时再补：

- [ ] sherpa-onnx Kotlin API 调用样例（参考 [官方 SherpaOnnxVadAsr demo](https://github.com/k2-fsa/sherpa-onnx/tree/master/android/SherpaOnnxVadAsr)）
- [ ] LlamaEngine.kt（项目自命名）的 C++ JNI wrapper 实现（参考 [llama.android](https://github.com/ggml-org/llama.cpp/tree/master/examples/llama.android) 官方模块 + [ToolNeuron `InferenceService.kt` + `InferenceClient.kt`](https://github.com/Siddhesh2377/ToolNeuron)（位于 `service/inference/` 目录））
- [ ] ForegroundService 通知的具体文案
- [ ] 错误重试的具体退避策略
- [ ] Compose UI 各屏幕的 wireframe
- [ ] 多语言资源（解决 G5 i18n，Phase 10 设计稿）
- [ ] 无障碍 contentDescription（解决 G6 a11y，Phase 10 设计稿）
- [ ] 音频路由（蓝牙/扬声器/听筒）切换逻辑（解决 G9，Phase 10 设计稿）
