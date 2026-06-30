# MiBrain - 红米 K50U 本地 AI 助手 项目设计文档

> 本文档**仅做设计，不交付代码**。先想清楚再动手。
>
> **2026-06-30 重大修订**（深度检查后第二轮调整）：
> - 推理后端从 llama-server HTTP 切回 JNI（[D7](../DECISIONS.md)），详见 [X7 废弃](../DECISIONS.md)
> - 模型路径改 DE 加密区 + Direct Boot（[D21](../DECISIONS.md)）
> - 默认模型从 3B 改 1.5B（[D1](../DECISIONS.md)）
> - 唤醒词改 sherpa-onnx KWS，弃用 openWakeWord（[D23](../DECISIONS.md)）
> - ASR/TTS 模型换 Apache 2.0 许可（[D22](../DECISIONS.md)）
> - 详细运行时细节见 [03_architecture_detail.md](./03_architecture_detail.md)

---

## 0. 项目元信息

| 项 | 值 |
|---|---|
| 项目名 | MiBrain（已确认，见 [D4](../DECISIONS.md)） |
| 目标设备 | 红米 K50 Ultra（骁龙 8+ Gen 1，8GB RAM，Adreno 730，HyperOS 3） |
| Root 方案 | KernelSU + ZygiskNext + LSPosed |
| 用户硬约束 | 只能保证 KSU + ZygiskNext + LSP，其他能力不保证（不自编译内核、不改 ROM、不长期维护复杂 hook） |
| 核心目标 | 离线、本地、隐私不出手机的语音助手 |
| 开发语言 | Kotlin（APK + JNI wrapper）+ Shell（KSU 脚本） |
| 当前阶段 | Phase 0：设计冻结（不交付代码，见 [README.md](../README.md)） |
| 推理后端 | llama.cpp b9830 + 官方 [llama.android](https://github.com/ggml-org/llama.cpp/tree/master/examples/llama.android) JNI 模块（[D7](../DECISIONS.md)） |
| 模型存储 | `/data/user_de/0/com.mibrain/files/models/`（DE 加密区 + Direct Boot，[D21](../DECISIONS.md)） |

---

## 1. 为什么要做这个项目（用户视角的问题陈述）

### 1.1 痛点

1. **商业助手体验差**：小爱、Siri 在国内受限于云识别 + 隐私顾虑
2. **本地 LLM 没有易用入口**：HostAI、Termux+llama.cpp 都需要懂技术
3. **MIUI/HyperOS 后台录音限制狠**：通用 APK 装上锁屏就哑
4. **现有方案要么太重要么太碎**：ToolNeuron 全功能但重；自己拼 HostAI+Tasker+PhantomMic 太碎没人维护

### 1.2 目标用户体验

> "我说『小脑』，手机在 3 秒内开始回答我的问题，全程离线。"

具体可量化的验收标准：

| 指标 | 目标 | 备注 |
|---|---|---|
| 唤醒响应延迟 | < 1s | 唤醒词检出 → 开始录音 |
| 录音 → 文本 | < 3s | 5 秒以内的短句 |
| 模型首 token | < 2s | 模型已 keep-alive 状态 |
| 首句语音回应 | < 5s | 含 TTS 合成 |
| 锁屏后可用率 | > 90% | 24 小时内随机测试 |
| 内存峰值 | < 7.6GB | 默认 1.5B 模型串行峰值约 7.5GB（[03 §6](./03_architecture_detail.md) 第三轮重算），留 0.5GB headroom（紧张，Stage 5 真机硬关卡） |
| 隐私 | 100% 离线 | 任何数据不上云 |

---

## 2. 与现有开源项目的差异化定位

### 2.1 调研的参考项目

| 项目 | 仓库 | 借鉴价值 | 不直接用的原因 |
|---|---|---|---|
| **ToolNeuron** | [Siddhesh2377/ToolNeuron](https://github.com/Siddhesh2377/ToolNeuron) (429★, 927 commits, Kotlin+Compose 全栈) | 完整 Kotlin 架构、GGUF 加载、TTS/STT/RAG 全套、`InferenceService.kt` + `InferenceClient.kt`（位于 `service/inference/` 目录）JNI 调用范式 | 太重，927 commits 学习成本高；未针对 MIUI 优化；无 LSPosed hook；但作为**代码参考样板**强烈推荐（[X2 重新评估](../DECISIONS.md)） |
| **HostAI** | wannaphong/android-hostai (v1.0.4, alpha) | LiteRT-LM + GPU 加速 + OpenAI API | 只解决"模型服务"一层，无语音、无 hook |
| **Phantom Mic** | [Xposed-Modules-Repo/tn.amin.phantom_mic](https://github.com/Xposed-Modules-Repo/tn.amin.phantom_mic) (LSPosed 官方模块仓库) | native hook AudioRecord.cpp，成熟 | 只 hook 不提供调度，用户自行安装 |
| **WhatsMicFix** | D4vRAM369/WhatsMicFix-LSPosed (75 commits) | LSPosed 双 scope 架构（app+system） | 专为 WhatsApp，需改造 |
| **XAudioCapture** | wzhy90/XAudioCapture | hook `PRIVATE_FLAG_ALLOW_AUDIO_PLAYBACK_CAPTURE` | 只解决"播放录音"非"麦克风录音" |
| **tasker-mcp** | cj-elevate/tasker-mcp | Go 交叉编译 arm64、Tasker HTTP 协议 | 是 MCP server，方向不同 |
| **Hotword Plugin + Snowboy** | jolanrensen | Tasker 唤醒词插件 + 自训 .umdl 模型 | Snowboy 已半弃维 |
| **wyoming-satellite-termux** | pantherale0 | Termux 跑本地唤醒 | 依赖 Termux（用户已排除） |
| **sherpa-onnx 官方示例** | [k2-fsa/sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx) (v1.13.3) | ASR/TTS/VAD/KWS 全栈 + Android Kotlin demo | 仅作为依赖，不作参考样板 |
| **llama.android** | [ggml-org/llama.cpp/tree/master/examples/llama.android](https://github.com/ggml-org/llama.cpp/tree/master/examples/llama.android) | 官方 JNI 模块（llama.cpp 主仓库内） | 直接作为依赖 |

### 2.2 我们的差异化

**MiBrain 的"窄而深"定位**：

- ❌ 不做：图像生成、RAG 知识库、插件系统（这些 ToolNeuron 已做得很好，用户可以直接装 ToolNeuron 补这些）
- ✅ 专注做：**MIUI/HyperOS 上的稳定后台语音对话**，这是 ToolNeuron 没解决的真问题
- ✅ 交付物：一个 KSU 模块 + 一个极简 APK + 配套 LSPosed 配置 + 部署脚本

**用一句话定义**：
> MiBrain 是一个针对小米 HyperOS 的"本地 LLM 语音助手"开箱即用方案，把"模型服务 + 语音 IO + 后台保活"三件事打包到一个 KSU 模块里，刷入即用。

---

## 3. 借鉴清单（具体到模块）

| MiBrain 模块 | 借鉴自 | 借鉴点 | 是否直接 fork |
|---|---|---|---|
| APK 主体 | ToolNeuron (re-write 分支) | 项目骨架、Gradle 配置、SAF 文件选择 | 否，参考重写 |
| LLM 推理 | [llama.android 官方 JNI 模块](https://github.com/ggml-org/llama.cpp/tree/master/examples/llama.android) + [ToolNeuron `InferenceService.kt` + `InferenceClient.kt`](https://github.com/Siddhesh2377/ToolNeuron)（位于 `service/inference/` 目录） | JNI wrapper 范式、模型加载/卸载、流式 callback | 参考样板（[D7](../DECISIONS.md)、[X2](../DECISIONS.md)） |
| TTS / STT / VAD / KWS | [sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx) v1.13.3 官方 Kotlin API | 一个 AAR 同时覆盖四能力，含 Android Kotlin demo | 直接依赖（[D8](../DECISIONS.md)、[D23](../DECISIONS.md)） |
| LSPosed 后台录音 | [Phantom Mic](https://github.com/Xposed-Modules-Repo/tn.amin.phantom_mic) (LSPosed 官方模块仓库) | native 层 AudioRecord.cpp hook | 装现成，不自写（[D9](../DECISIONS.md)） |
| KSU 模块壳 | 标准 KSU 模板 | post-fs-data.sh / sepolicy.rule / appops | 直接用 |

> **废弃路径**（详见 [DECISIONS.md X1-X7](../DECISIONS.md)）：
> - ~~Go daemon~~（X1，CGO + NDK 交叉编译门槛高）
> - ~~llama-server HTTP 路径~~（X7，深度检查发现无 Android 先例 + S1/S2/S3 根因问题；切回 JNI，见 [D7](../DECISIONS.md)）
> - ~~自写 LSPosed hook~~（X5，Phantom Mic 已完整覆盖 native hook）
> - ~~WhatsMicFix 改造~~（专为 WhatsApp，Phantom Mic 已够用）
> - ~~openWakeWord 自写 Kotlin wrapper~~（[D23](../DECISIONS.md)，sherpa-onnx 已内置 KWS，统一技术栈）
>
> **澄清**：~~fork ToolNeuron 推理封装~~（[X2 原误判已修正](../DECISIONS.md)，实际 ToolNeuron 是 Kotlin+Compose 全栈，真实推理封装为 `InferenceService.kt` + `InferenceClient.kt`（位于 `service/inference/` 目录），可作为参考样板，不再算"废弃"）

---

## 4. 关键设计决策（带依据）

### 4.1 决策：用 Kotlin APK 而非 Go daemon（推翻之前的方案）

**原因**：
- 之前选 Go daemon，调研发现 Android 上 `CGO_ENABLED=0` DNS 会崩，必须 NDK 交叉编译，对用户门槛太高
- ToolNeuron 用 Kotlin + llama.cpp JNI 已经验证可行，927 commits 跑通
- 单 APK 比"APK + Go daemon"两层简单
- 8GB 内存下多一个 Go daemon 进程是负担

**结果**：所有逻辑都在 Kotlin APK 里，KSU 模块只负责"放二进制 + 改系统设置"

### 4.2 决策：llama.cpp 而非 LiteRT-LM（2026-06-30 修订：切回 JNI）

**原因**：
- LiteRT-LM（HostAI 用的）只支持 .litertlm 格式，模型选择面窄
- llama.cpp 支持 GGUF，社区模型海量（Qwen、Phi、Gemma 全有）
- llama.cpp 官方提供 [llama.android](https://github.com/ggml-org/llama.cpp/tree/master/examples/llama.android) JNI 模块，已在 PocketPal/ToolNeuron 等多个 Android LLM 项目中验证可用
- GPU 加速：Adreno 730 支持 Vulkan，未来可启用 `-DLLAMA_VULKAN=ON`；OpenCL + Adreno 后端**无 Android 预编译包**（仅 Windows arm64 版 `llama-b9830-bin-win-opencl-adreno-arm64.zip`），Android 需用 Snapdragon 工具链 Docker 镜像自编译（`GGML_OPENCL=ON` + `GGML_HEXAGON=ON`），Phase 0-5 不做 GPU 加速

**结果**：APK 内通过 JNI 调用 libllama.so，参考 [llama.android 官方模块](https://github.com/ggml-org/llama.cpp/tree/master/examples/llama.android) + ToolNeuron `InferenceService.kt` + `InferenceClient.kt`（位于 `service/inference/` 目录）（[D7](../DECISIONS.md)、[X2](../DECISIONS.md)）。**不再走 llama-server HTTP 路径**（[X7 废弃](../DECISIONS.md)）

### 4.3 决策：sherpa-onnx 做 TTS + STT + VAD + KWS（2026-06-30 修订：全栈统一）

**原因**：
- sherpa-onnx v1.13.3 一个 AAR 同时提供 ASR / TTS / VAD / KWS 四能力，省依赖
- sherpa-onnx 官方提供 Android Kotlin demo，已验证可行
- 内置 VAD，省一层依赖
- 内置 KWS（keyword spotting），替代 openWakeWord（[D23](../DECISIONS.md)）

**结果**：一套 sherpa-onnx 解决四件事，体积小、依赖少、技术栈统一

### 4.4 决策：LSPosed 用 Phantom Mic 的 native hook（2026-06-30 修订：澄清风险）

**原因**：
- MIUI 后台录音限制在 Java 层 hook 不够，必须 native
- Phantom Mic 已 hook `AudioRecord.cpp` native 层，覆盖 95% App
- 来自 [LSPosed 官方模块仓库](https://github.com/Xposed-Modules-Repo/tn.amin.phantom_mic)
- 我们不需要重写 hook，只要在 LSPosed 配置里启用即可

**结果**：不自己写 LSPosed 模块，**装现成的 Phantom Mic**，只在文档里给配置指引

> ⚠️ **风险升级（见 [D14](../DECISIONS.md)）**：Phantom Mic v2.0 发布于 2024-07，至本次设计冻结（2026-06-30）**已近 2 年未更新**，**第四轮 web 核实确认上游已停滞 23 个月**（自 2024-07-24 v2.0 后零更新），HyperOS 3（Android 15）兼容性未验证且无官方更新。原稿"活跃维护"判断已修正为误判。Phase 1 已确认无新版本；降级方案见 D14（appops + 双触发兜底 / WhatsMicFix 改造 / 放弃锁屏唤醒）

### 4.5 决策：不依赖 Tasker，APK 自带语音对话

**原因**：
- Tasker + AutoVoice 在国内依赖 Google 服务，不稳
- Tasker HTTP Request 不支持 SSE 流式
- 用户已说"APK 部分要完整模板"，那就一次性做到位

**结果**：APK 自己实现录音 → 唤醒 → ASR → LLM → TTS → 播放完整链路

---

## 5. 整体架构

```
┌────────────────────────────────────────────────────────────────────┐
│  Android 系统层（HyperOS 3）                                        │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ LSPosed 注入（root 后启动）                                   │   │
│  │  └─ Phantom Mic (现成) → hook AudioRecord.cpp native 层      │   │
│  │     → 让 APK 即使在后台/锁屏也能持续拿到麦克风数据             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              ↑                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ MiBrain APK (Kotlin + JNI, uid=10xxx, 非 root)               │   │
│  │  ┌────────────────────────────────────────────────────────┐  │   │
│  │  │ 前台服务 ForegroundService (microphone type)            │  │   │
│  │  │  ├─ directBootAware=true（锁屏前可启动，[D21](../DECISIONS.md)）│   │
│  │  │  ├─ AudioRecord 16kHz mono int16 (持续)                │  │   │
│  │  │  ├─ sherpa-onnx VAD (实时检测说话段)                   │  │   │
│  │  │  ├─ sherpa-onnx KWS (唤醒词检出，替代 openWakeWord)     │  │   │
│  │  │  └─ 检测到唤醒 → 切换到对话模式（CAS 原子转换）          │  │   │
│  │  └────────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────────┐  │   │
│  │  │ 对话引擎 ConversationEngine                            │  │   │
│  │  │  ├─ sherpa-onnx 流式 ASR (说话→文本)                   │  │   │
│  │  │  ├─ LlamaEngine.kt (JNI 调用 libllama.so，流式 callback)│  │   │
│  │  │  ├─ sherpa-onnx 流式 TTS (文本→语音，按句播放)         │  │   │
│  │  │  └─ AudioTrack 播放队列                                 │  │   │
│  │  └────────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────────┐  │   │
│  │  │ UI 层 (Jetpack Compose)                                │  │   │
│  │  │  ├─ 主界面：状态显示、对话气泡                          │  │   │
│  │  │  ├─ 设置：模型选择、唤醒词、TTS 声音、温度、联网开关    │  │   │
│  │  │  └─ 模型管理：下载、删除、激活（下载到 DE 区）          │  │   │
│  │  └────────────────────────────────────────────────────────┘  │   │
│  │                                                              │   │
│  │  JNI 库 (jniLibs/arm64-v8a/)：                              │   │
│  │   ├─ libllama.so + libggml.so   ← llama.cpp b9830 编译产出   │   │
│  │   ├─ libsherpa-onnx-jni.so      ← sherpa-onnx v1.13.3 AAR    │   │
│  │   └─ libonnxruntime.so          ← sherpa-onnx AAR 自带        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ KernelSU 模块 mi_brain_module                                │   │
│  │  ├─ post-fs-data.sh：                                        │   │
│  │  │   ├─ appops set <uid> RECORD_AUDIO allow                 │   │
│  │  │   ├─ appops set <uid> OP_POST_NOTIFICATIONS allow（前台通知）│   │
│  │  │   └─ 创建日志目录 + chcon SELinux 上下文                 │   │
│  │  ├─ sepolicy.rule：放行 Phantom Mic hook 所需 SELinux 类型   │   │
│  │  ├─ system/etc/sysconfig/mibrain.xml：电池白名单             │   │
│  │  └─ ⚠️ 不再放 llama-server 二进制（JNI 路径下不需要）        │   │
│  └─────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────┘
                              ↑
                              │ APK 首次启动时下载到 DE 加密区
                              │ /data/user_de/0/com.mibrain/files/models/
                              │ （Direct Boot 下可读，[D21](../DECISIONS.md)）
                              │
┌────────────────────────────────────────────────────────────────────┐
│ 模型资产（不打包进模块与 APK，APK 内置下载器引导用户下载）           │
│  ├─ qwen2.5-1.5b-instruct-q4_k_m.gguf   (~1GB，默认，[D1](../DECISIONS.md))│
│  ├─ qwen2.5-3b-instruct-q4_k_m.gguf     (~2GB，质量优先可选)       │
│  ├─ sherpa-onnx-streaming-zipformer-bilingual-zh-en (流式 ASR，Apache 2.0)│
│  ├─ sherpa-onnx-vits-zh-ll (TTS，~150MB，社区贡献许可未明确，[D22](../DECISIONS.md))│
│  ├─ silero_vad.onnx (VAD，~10MB，sherpa-onnx 自带)                 │
│  └─ hey_jarvis.onnx (KWS，~10MB，[D23](../DECISIONS.md))           │
└────────────────────────────────────────────────────────────────────┘
```

> **架构变更说明（2026-06-30 修订，第二轮深度检查后）**：
> - **推理后端从 llama-server HTTP 切回 JNI**（[D7](../DECISIONS.md)）：APK 内通过 JNI 直接调用 libllama.so，不再有 root 启动的 llama-server 子进程，不再有 127.0.0.1:8080 HTTP 端口（除非 Phase 8 Cap 1 启用本地 API 暴露）
> - **模型路径改 DE 加密区 + Direct Boot**（[D21](../DECISIONS.md)）：从 `/sdcard/MiBrain/models/` 改为 `/data/user_de/0/com.mibrain/files/models/`，解决 FBE 加密锁屏读不到模型 + 跨 SELinux 域两大根因问题
> - **KSU 模块职责瘦身**：不再放 llama-server 二进制，不再写 service.sh watchdog，只负责 appops + SELinux + 电池白名单
> - 详见 [DECISIONS.md D7 / D21 / X7](../DECISIONS.md) 与 [03_architecture_detail.md](./03_architecture_detail.md)

---

## 6. 组件选型与版本

| 组件 | 选型 | 版本 | 来源 |
|---|---|---|---|
| LLM 推理 | llama.cpp + 官方 [llama.android](https://github.com/ggml-org/llama.cpp/tree/master/examples/llama.android) JNI 模块 | b9830 | github.com/ggml-org/llama.cpp |
| LLM JNI wrapper | LlamaEngine.kt（项目自命名；实现参考 ToolNeuron `InferenceService.kt` + `InferenceClient.kt`，[X2](../DECISIONS.md)） | - | 自写 ~1500 行 Kotlin + C，参考样板 |
| 流式 ASR | sherpa-onnx (官方 Kotlin API) | v1.13.3 | github.com/k2-fsa/sherpa-onnx |
| 流式 TTS | sherpa-onnx VITS | v1.13.3 | 同上 |
| VAD | sherpa-onnx silero_vad | v1.13.3 | 同上（一个 AAR 覆盖） |
| 唤醒词 KWS | sherpa-onnx kws-zipformer-wenetspeech-3.3M-2024-01-01 | v1.13.3 | 同上（[D23](../DECISIONS.md)，弃用 openWakeWord；原引用 `zh-vgg` 不存在，真实模型为 zipformer 架构） |
| LSPosed hook | [Phantom Mic](https://github.com/Xposed-Modules-Repo/tn.amin.phantom_mic) | 2.0 | LSPosed 官方模块仓库 |
| UI 框架 | Jetpack Compose | BOM 2024.06+ | Google |
| 异步 | Kotlin Coroutines | 1.8+ | JetBrains |
| 持久化 | Room | 2.6+ | AndroidX |
| KSU 模块壳 | 标准 KSU 模板 | - | kernelsu.org |
| Gradle | Kotlin DSL | 8.5+ | - |

> **2026-06-30 修订说明**：
> - 删除原"Android arm64 二进制 | llama-b9830-bin-android-arm64"条目（不再走 HTTP 路径，KSU 模块不再放 llama-server 二进制）
> - LLM 推理改为 JNI 调用 libllama.so + libggml.so（APK 内嵌 jniLibs/arm64-v8a/）
> - 唤醒词从 openWakeWord 改为 sherpa-onnx KWS（统一技术栈，[D23](../DECISIONS.md)）
> - VAD 单列条目（原隐含在 ASR 里），明确为 sherpa-onnx 自带 silero_vad
> - 详细 AAR 内部结构见 [01_feasibility_verification.md](./01_feasibility_verification.md) §2.2

---

## 7. 数据流时序（一次完整对话）

```
用户              APK 前台服务          sherpa-onnx       LlamaEngine(JNI)  AudioTrack
 │                   │                    │                  │                 │
 │ "嘿小脑"          │                    │                  │                 │
 ├──────────────────►│                    │                  │                 │
 │                   │ 唤醒词检出 (80ms)  │                  │                 │
 │                   │ state=LISTENING    │                  │                 │
 │                   │ acquire WakeLock   │                  │                 │
 │                   │ ─────────────────►│ KWS              │                 │
 │                   │ ◄─────────────────┤ detected=true    │                 │
 │                   │                    │                  │                 │
 │ 提示音"嗯？"       │                    │                  │                 │
 │ ◄─────────────────┤                    │                  │                 │
 │                   │ state=LISTENING     │                  │                 │
 │                   │ ─────────────────►│ ASR              │                 │
 │ "明天会下雨吗"     │                    │                  │                 │
 ├──────────────────►│ AudioRecord        │                  │                 │
 │                   │ ─────────────────►│ VAD + ASR        │                 │
 │                   │ ◄─────────────────┤ "明天会下雨吗"   │                 │
 │                   │                    │                  │                 │
 │                   │ state=THINKING     │                  │                 │
 │                   │ streamComplete(prompt)                  │                 │
 │                   │ ─────────────────────────────────────►│ llama_decode    │
 │                   │ ◄──callback token────────────────────┤ "明天"          │
 │                   │ ◄──callback token────────────────────┤ "明天晴"        │
 │                   │ 切句检测：检测到"。"                    │                 │
 │                   │ state=SPEAKING     │                  │                 │
 │                   │ ─────────────────►│ TTS "明天晴"     │                 │
 │                   │ ◄─────────────────┤ audio chunk       │                 │
 │ 听到"明天晴"       │ ─────────────────────────────────────────────────────►│ AudioTrack
 │ ◄─────────────────┤                    │                  │                 │
 │                   │ 播放完毕            │                  │                 │
 │                   │ state=COOLDOWN(500ms)│                │                 │
 │                   │ state=IDLE         │                  │                 │
 │                   │ release WakeLock   │                  │                 │
 │                   │ 重新激活唤醒检测     │                  │                 │
 │                   │ ... 继续流式 ...    │                  │                 │
```

**关键设计点**：
- **状态机 CAS 原子转换**（解决并发抢占，详见 [03_architecture_detail.md §4](./03_architecture_detail.md)）：每个状态切换通过 `AtomicReference.compareAndSet`，避免唤醒/通知/工具同时触发
- **句子切片阈值**：检测 `[。！？!?\n]` 任一符号即切句送 TTS，不等待整段生成完
- **回环防护**：SPEAKING + COOLDOWN 期间丢弃所有麦克风数据
- **超时降级**：若 5s 无 token，先 TTS"让我想想..."安抚用户
- **WakeLock**：IDLE→LISTENING 时 acquire，COOLDOWN→IDLE 时 release（详见 [03_architecture_detail.md §4 WakeLock](./03_architecture_detail.md)）

---

## 8. 仓库结构设计

### 8.1 单仓库 vs 多仓库

**单仓库**：所有组件放一起，方便用户一个 clone 拿全部。

### 8.2 仓库目录结构

```
mibrain/                                    # 仓库根
├── README.md                              # 项目介绍 + 快速开始
├── LICENSE                                # Apache 2.0
├── CONTRIBUTING.md                        # 贡献指南
├── docs/                                  # 文档（本设计文档在此）
│   ├── 00_design_overview.md              # 本文档（完整设计）
│   ├── 01_feasibility_verification.md     # 可行性验证报告（依赖可达）
│   ├── 02_second_review.md                # 第二轮严谨审视
│   ├── 03_architecture_detail.md          # 详细架构与时序图（JNI 接口、状态机 CAS、内存预算）
│   ├── 04_build_guide.md                  # 编译指南 ⏳ Phase 1
│   ├── 05_deploy_guide.md                 # 部署指南（用户向）
│   ├── 06_lspoded_setup.md                # LSPosed 配置指南（Phantom Mic 详解）
│   ├── 07_troubleshooting.md              # 故障排查
│   ├── 08_performance_bench.md            # 性能基准 ⏳ Phase 5
│   ├── 09_phase6_network_tools_design.md  # Phase 6 联网工具设计稿
│   ├── 10_phase7_phone_control_design.md  # Phase 7 手机控制设计稿
│   ├── 11_phase8_platform_design.md       # Phase 8 平台化设计稿
│   ├── 12_phase9_multimodal_design.md     # Phase 9 多模态设计稿
│   └── 13_phase10_ux_enhancements_design.md # Phase 10 用户体验增强设计稿
│
├── app/                                   # MiBrain APK（Android Studio 项目）
│   ├── app/
│   │   ├── build.gradle.kts
│   │   ├── src/main/
│   │   │   ├── AndroidManifest.xml        # directBootAware=true
│   │   │   ├── java/com/mibrain/
│   │   │   │   ├── MainActivity.kt         # Compose 入口
│   │   │   │   ├── service/
│   │   │   │   │   ├── MiBrainForegroundService.kt   # 前台服务（directBootAware）
│   │   │   │   │   └── AudioPipeline.kt            # 录音→VAD→KWS→ASR
│   │   │   │   ├── engine/
│   │   │   │   │   ├── LlamaEngine.kt              # JNI 调用 libllama.so（项目自命名，参考 ToolNeuron `InferenceService.kt` + `InferenceClient.kt`）
│   │   │   │   │   ├── SherpaAsrEngine.kt          # 流式 ASR
│   │   │   │   │   ├── SherpaTtsEngine.kt          # 流式 TTS
│   │   │   │   │   ├── SherpaVadEngine.kt          # VAD（silero_vad）
│   │   │   │   │   ├── SherpaKwsEngine.kt          # 唤醒词 KWS（替代 openWakeWord）
│   │   │   │   │   ├── ConversationEngine.kt      # 编排 + 状态机 CAS
│   │   │   │   │   └── WakeLockManager.kt         # WakeLock 管理
│   │   │   │   ├── jni/                           # JNI native 源码
│   │   │   │   │   ├── CMakeLists.txt             # CMake 配置
│   │   │   │   │   ├── llama_engine.cpp           # LlamaEngine.kt 对应的 JNI 实现
│   │   │   │   │   └── llama_engine.h
│   │   │   │   ├── data/
│   │   │   │   │   ├── ModelManager.kt            # 模型管理（运行时下载到 DE 区）
│   │   │   │   │   ├── SettingsRepository.kt      # 读 config.json
│   │   │   │   │   └── db/                        # Room
│   │   │   │   ├── ui/
│   │   │   │   │   ├── theme/
│   │   │   │   │   ├── screens/                    # 各 Compose 屏幕
│   │   │   │   │   └── components/
│   │   │   │   └── util/
│   │   │   ├── assets/
│   │   │   │   └── default_prompts.json           # 默认 system prompt
│   │   │   └── jniLibs/arm64-v8a/                  # 预编译 .so
│   │   │       ├── libllama.so                     # llama.cpp b9830 编译产出
│   │   │       ├── libggml.so                      # llama.cpp b9830 编译产出
│   │   │       ├── libsherpa-onnx-jni.so           # sherpa-onnx v1.13.3 AAR
│   │   │       └── libonnxruntime.so               # sherpa-onnx AAR 自带
│   │   └── proguard-rules.pro
│   ├── settings.gradle.kts
│   └── gradle/libs.versions.toml                    # 版本目录
│
├── ksu_module/                            # KSU 模块源码
│   ├── module.prop
│   ├── install.sh
│   ├── post-fs-data.sh                   # appops + chcon SELinux + 电池白名单
│   ├── uninstall.sh
│   ├── sepolicy.rule                     # 放行 Phantom Mic hook 所需类型
│   ├── system/etc/sysconfig/mibrain.xml  # 电池白名单
│   └── build.sh                          # 打包 zip 脚本
│   # ⚠️ 不再有 service.sh（无 llama-server 子进程）
│   # ⚠️ 不再有 libs/llama-server 二进制目录
│
├── scripts/                               # 辅助脚本
│   ├── download_models.sh                 # 一键下载模型到 DE 区（HF + 阿里 OSS 镜像）
│   ├── build_llama_jni.sh                 # 编译 llama.cpp b9830 → libllama.so + libggml.so
│   ├── verify_assets.sh                   # 校验所有下载文件的 SHA256
│   ├── build_ksu_zip.sh                   # 打包 KSU 模块 zip
│   └── ci/                                # CI 脚本 ⏳ Phase 5
│       ├── build_apk.sh                   # CI 中构建 APK（含 JNI 编译）
│       └── verify_design.sh              # CI 中验证设计文档完整性
│   # ⚠️ 不再有 download_binaries.sh（llama-server 二进制路径已废弃）
│
├── .github/                               # GitHub 配置
│   ├── ISSUE_TEMPLATE/                    # Issue 模板（bug / feature / test_report）
│   └── workflows/                         # GitHub Actions ⏳ Phase 5
│       ├── build_apk.yml                  # 自动构建 APK
│       ├── build_ksu_module.yml           # 自动打包 KSU 模块
│       └── release.yml                    # 发布 release
│
└── third_party/                           # 第三方资源引用清单
    ├── NOTICES.md                         # 所有引用的开源项目
    └── licenses/                          # 完整许可文本 ⏳ Phase 1 补充
```

> **2026-06-30 修订说明**：
> - `LlamaHttpClient.kt` → `LlamaEngine.kt`（HTTP 调用改 JNI 调用）
> - 新增 `app/src/main/java/com/mibrain/jni/` 目录放 C++ JNI 实现
> - 新增 `WakeLockManager.kt`、`SherpaVadEngine.kt`、`SherpaKwsEngine.kt`（原 WakeWordEngine.kt 拆分）
> - 删除 `ksu_module/libs/llama-server` 二进制目录
> - 删除 `ksu_module/service.sh`（无子进程需要启动）
> - `scripts/download_binaries.sh` → `scripts/build_llama_jni.sh`（自编译 libllama.so）
> - `AndroidManifest.xml` 加 `directBootAware=true`
> - docs/ 补全 Phase 6-9 四份设计稿

---

## 9. 模块职责矩阵

| 模块 | 职责 | 失败时的降级 |
|---|---|---|
| BrainForegroundService | 持续运行、保活、协调各 engine | 重启自身（watchdog） |
| AudioPipeline | 录音 + VAD + 唤醒词 | 显示"麦克风异常" toast |
| WakeWordEngine | 检测唤醒词 | 切换到桌面快捷方式手动触发 |
| SherpaAsrEngine | 流式语音转文本 | 提示用户重说 |
| LlamaEngine | LLM 流式生成 | 提示"模型加载失败" |
| SherpaTtsEngine | 文本转语音 | 退回 Android 内置 TTS |
| ConversationEngine | 编排上述四者 | 单点失败即停止当前对话 |
| ModelManager | 模型文件管理 | 缺失模型时引导下载 |
| SettingsRepository | 配置持久化 | 用默认配置 |
| UI 层 | 用户交互 | - |

---

## 10. 风险与缓解

| 风险 | 概率 | 影响 | 缓解策略 |
|---|---|---|---|
| 8GB 内存峰值触发 lmkd | 高 | 高 | 默认 1.5B 模型（[D1](../DECISIONS.md)），串行峰值约 7.5GB / 最坏叠加 7.89GB（[03 §6](./03_architecture_detail.md) 第三轮重算）；模型 keep-alive 5min 自动卸载；监听 `onTrimMemory(TRIM_MEMORY_RUNNING_CRITICAL)` 主动调 `unloadModel()`（VAD+KWS 不卸载）；Stage 5 真机 24h 硬关卡 |
| MIUI 锁屏杀后台 | 高 | 高 | Phantom Mic hook + sysconfig 白名单 + 前台服务通知 + Direct Boot（[D21](../DECISIONS.md)） |
| sherpa-onnx native 库体积大 | 中 | 中 | 用 proguard 移除未用 ABI；只打包 arm64-v8a |
| 唤醒词误触发 | 中 | 低 | 双段确认（唤醒后说"嗯？"再听命令）；KWS cooldown 5s（[03 §8 config.json](./03_architecture_detail.md)） |
| **Phantom Mic 上游停滞 / HyperOS 3 不兼容** | **高** | **高** | 见 [DECISIONS.md D14](../DECISIONS.md)；**第四轮 web 核实确认上游已停滞 23 个月**（自 2024-07-24 v2.0 后零更新）；Phase 1 前已确认无新版本；降级方案：appops + 双触发兜底 / WhatsMicFix 改造 / 放弃锁屏唤醒（详见 D14） |
| llama.cpp 升级破坏 JNI ABI | 中 | 中 | 锁版本到 b9830（[D7](../DECISIONS.md)，第四轮 [F3](./14_feasibility_recheck_and_plan.md) 订正：原 b9844 未发布）；CI 自动验证 JNI 接口签名；`LlamaEngine.getVersion()` 启动时校验（[03 §3.3](./03_architecture_detail.md)） |
| 麦克风被其他 App 占用 | 低 | 中 | 监听 AudioFocus 变化；占用时静默等待 |
| JNI wrapper 工程量 ~1500 行被低估 | 中 | 中 | 参考 [ToolNeuron `InferenceService.kt` + `InferenceClient.kt`](https://github.com/Siddhesh2377/ToolNeuron)（位于 `service/inference/` 目录）（[X2](../DECISIONS.md)）；Phase 1 优先打通最小路径，复杂特性（function calling）后置 |
| 模型下载到 DE 区失败 / SHA256 不匹配 | 低 | 中 | APK 内置下载器带断点续传 + SHA256 校验 + 多源切换（HF 主 → 阿里 OSS 镜像兜底）；失败删除重试 |
| **TTS 无流式 chunk 回调** | **中** | **中** | sherpa-onnx TTS 是 OfflineTts（一次性生成完整 PCM 数组），无官方流式回调；MVP 用 "先合成完整 PCM 再播" 模式（[D26](../DECISIONS.md)）；Phase 2 / Phase 9 G6 字幕同步需在 Kotlin 端按 200ms 切 PCM 数组成 chunk 喂 AudioTrack 对齐播放进度 |
| **Qwen2.5-1.5B function calling 不稳** | **高** | **中** | BFCL v3 实测直答准确率仅 44%，32-token CoT 提升到 64%，长 CoT 反崩到 25%；Phase 6 工具调用必须用 32-token CoT prompt 或换 3B 模型（[D25](../DECISIONS.md)）；[D17](../DECISIONS.md) 混合方案关键词正则优先兜底 |
| **Direct Boot + sherpa-onnx 加载未公开验证** | **中** | **高** | sherpa-onnx 能否在 Direct Boot 下加载 .so + ONNX 模型无先例（BCR 仅验证 Direct Boot + AudioRecord）；Stage 1 PoC-A 设为硬关卡，4 子项（.so 加载 / ONNX 模型加载 / ORT 无 SharedPreferences 副作用 / HyperOS 3 不杀 FGS）任一失败则放弃 Direct Boot 退到"用户首次解锁后才启动"（[D27](../DECISIONS.md)） |

---

## 11. 交付路线图

### Phase 0：设计冻结（当前）
- ✅ 本文档完成（第二轮深度检查后修订完成）
- ✅ [DECISIONS.md](../DECISIONS.md) 决策清单（31 已确认 + 7 已废弃，第四轮 web 核实后修订）
- ✅ [03_architecture_detail.md](./03_architecture_detail.md) 详细架构（JNI 接口、状态机 CAS、内存预算、Direct Boot 流程）
- ✅ Phase 6-9 四份扩展设计稿 + Phase 10 一份 UX 增强设计稿
- ⏳ 等待用户最终审阅

### Phase 1：MVP 单链路打通（最小可用）

> **技术细节直接依据**：本 Phase 涉及的全部技术规范（llama.cpp b9830 CMake 编译 + .so 输出路径 + jniLibs 结构 / JNI 跨线程 callback / b9830 C API 清单 / KSU 模块脚本完整草案）详见 [15_technical_specs.md](./15_technical_specs.md) §1-§4，Phase 1 编码前必读。

- [ ] Kotlin APK 骨架 + Compose UI（参考 [ToolNeuron](https://github.com/Siddhesh2377/ToolNeuron)）
- [ ] **JNI 集成**：编译 llama.cpp b9830 → libllama.so + libggml.so（参考 [llama.android 官方模块](https://github.com/ggml-org/llama.cpp/tree/master/examples/llama.android)）
- [ ] **LlamaEngine.kt**：JNI wrapper（项目自命名，参考 ToolNeuron `InferenceService.kt` + `InferenceClient.kt`，~1500 行 Kotlin + C）
- [ ] AndroidManifest.xml：`directBootAware=true` + ForegroundService(microphone)
- [ ] 一个静态对话界面（按按钮触发，非语音）
- [ ] ModelManager：DE 区模型下载 + SHA256 校验
- [ ] KSU 模块壳：post-fs-data.sh + sepolicy.rule + sysconfig 电池白名单（**无 service.sh**）
- **验收**：能通过 UI 跟模型对话（非语音），JNI 加载模型成功

### Phase 2：语音链路
- [ ] sherpa-onnx ASR 集成
- [ ] sherpa-onnx TTS 集成
- [ ] 前台服务 + AudioRecord + AudioTrack
- [ ] 对话编排（手动按钮触发录音）
- [ ] 句子切片阈值（检测 `。！？!?\n`）
- **验收**：按按钮说话，能听到回答

### Phase 3：唤醒词
- [ ] sherpa-onnx KWS 集成（[D23](../DECISIONS.md)，弃用 openWakeWord）
- [ ] 持续监听 + 切换对话模式
- [ ] 回环防护（状态机 + CAS，[03 §4](./03_architecture_detail.md)）
- [ ] WakeLock 管理（[03 §4 WakeLock](./03_architecture_detail.md)）
- **验收**：说"hey jarvis"自动启动对话

### Phase 4：稳定性
- [ ] Phantom Mic 配置文档
- [ ] KSU appops 自动配置
- [ ] 内存监控 + 自动卸载（监听 onTrimMemory）
- [ ] CI 自动构建
- [ ] **D14 复查**：Phase 1 启动前去 [上游仓库](https://github.com/Xposed-Modules-Repo/tn.amin.phantom_mic/releases) 复查 Phantom Mic 活跃度 + HyperOS 3 兼容性（第四轮 web 核实已确认上游停滞 23 个月，此项转为"确认降级方案可用性"）
- **验收**：锁屏 24h 后仍可唤醒；3B 模型 OOM 测试（[D1](../DECISIONS.md)）

### Phase 5：发布
- [ ] GitHub Release 自动化
- [ ] 完整文档
- [ ] 演示视频
- [ ] Issue 模板

### Phase 6：联网工具调用（设计草案 [09_phase6_network_tools_design.md](./09_phase6_network_tools_design.md)）
- [ ] 全局联网开关 + UI 设置
- [ ] Tool 接口 + ToolRouter + NetworkGate 基础设施
- [ ] 4 个联网工具：WeatherTool / TranslateTool / WebSearchTool / NewsTool
- [ ] 状态机扩展（新增 TOOL_RUNNING 状态）
- **验收**：开关打开后，说"明天天气"能联网查询并语音回答

### Phase 7：手机控制类（设计草案 [10_phase7_phone_control_design.md](./10_phase7_phone_control_design.md)）
- [ ] SystemSettingsTool（手电筒/亮度/静音等 10+ 设置）
- [ ] NotificationReaderTool（朗读 + 语音回复，**走 A2A 路线**，详见设计稿）
- [ ] AppLauncherTool L1（仅启动 app，**用 `monkey -p $pkg 1`**，[D19](../DECISIONS.md)）
- [ ] ScreenCaptureTool（截屏 + vision API，[D20](../DECISIONS.md)）
- **验收**：锁屏时来微信通知能朗读并语音回复

### Phase 8：平台化能力（设计草案 [11_phase8_platform_design.md](./11_phase8_platform_design.md)）
- [ ] Cap 1: 本地 API 暴露（端口配置 + 限流 + token 计数 + LAN 选项）
- [ ] Cap 3: 桌面小组件（一键对话 / 一键听写 / 状态卡）
- [ ] Cap 4: MCP 协议支持（暴露 8 个 MCP 工具给桌面端）
- **验收**：桌面端 Claude Desktop 能通过 MCP 调用手机上的 MiBrain 能力

### Phase 9：多模态（设计草案 [12_phase9_multimodal_design.md](./12_phase9_multimodal_design.md)）
- [ ] ImageChatTool（单轮 OCR + 多轮看图对话）
- [ ] 相机 intent + 图片 base64 处理
- [ ] 多轮上下文管理
- [ ] 状态机扩展（新增 IMAGE_CAPTURING 状态，含 COOLDOWN 过渡）
- **验收**：拍照后能多轮对话问图的内容

### Phase 10：用户体验增强（设计草案 [13_phase10_ux_enhancements_design.md](./13_phase10_ux_enhancements_design.md)）
- [ ] G1: 多用户（APK 内 profile 切换，最多 3 个，独立对话历史 + TTS/唤醒词配置）
- [ ] G2: 儿童模式（双层过滤 + 工具限制 + 时长限制 + PIN 切换）
- [ ] G5: i18n（中英双语 MVP，UI/ASR/TTS/LLM 分层切换）
- [ ] G6: a11y（视障 TalkBack + 听障字幕 + 状态颜色化）
- [ ] G9: 音频路由（蓝牙耳机/扬声器自动跟随系统）
- [ ] G17: 全局暂停（Widget + 通知按钮 + 双击电源键可选）
- **验收**：家庭 3 个 profile 切换不丢对话；蓝牙耳机能用；紧急情况能 1 秒暂停

---

## 12. 知识产权与许可

- **本项目许可**：Apache 2.0
- **必须保留的 NOTICES**：
  - llama.cpp (MIT)
  - sherpa-onnx (Apache 2.0)
  - onnxruntime (MIT)
  - Compose / AndroidX (Apache 2.0)
- **不打包的第三方资产**：
  - 模型文件（用户自下，全部选 Apache 2.0 许可，[D22](../DECISIONS.md)）
  - Phantom Mic APK（用户自装，LSPosed 模块，许可待 Phase 1 复核）
- **2026-06-30 修订**：
  - 删除原 "openWakeWord (MIT)" 条目（弃用 openWakeWord，[D23](../DECISIONS.md)）
  - ASR/TTS/VAD/KWS 全部用 sherpa-onnx 官方 Apache 2.0 许可模型，避免原 paraformer (CC BY-NC) + aishell3 (CC BY-NC-ND) 的许可冲突（[D22](../DECISIONS.md)）

---

## 13. 开放问题（全部已确认 + 二轮深度检查新增）

> **更新（2026-06-30）**：原 6 个开放问题已全部在 [DECISIONS.md](../DECISIONS.md) 中确认；深度检查后又新增 D21-D23 共 3 条已确认决策 + 1 条风险升级。

| # | 原问题 | 决策 | 决策编号 | 状态 |
|---|---|---|---|---|
| 1 | GitHub 仓库归属 | `qbjsdsb/mibrain` | [D12](../DECISIONS.md) | ✅ 已确认 |
| 2 | 项目名是否用 "MiBrain" | 是，确认 MiBrain | [D4](../DECISIONS.md) | ✅ 已确认 |
| 3 | 是否借鉴 ToolNeuron 代码结构 | 参考骨架 + `InferenceService.kt` + `InferenceClient.kt` JNI 范式（[X2 重新评估](../DECISIONS.md)，不再算废弃） | [X2](../DECISIONS.md) | ✅ 已确认 |
| 4 | 唤醒词定什么 | MVP 用英文 `hey_jarvis`（sherpa-onnx KWS），Phase 3 自训中文 | [D2](../DECISIONS.md)、[D23](../DECISIONS.md) | ✅ 已确认 |
| 5 | 默认模型选哪个 | **Qwen2.5-1.5B-Instruct Q4_K_M**（~1GB）；**3B 在 8GB 设备必 OOM 不可用**（[D30](../DECISIONS.md)），仅 12GB+ 或 Phase 11+ GPU 加速后可选 | [D1](../DECISIONS.md) + [D30](../DECISIONS.md) | ✅ 已确认 |
| 6 | 是否需要 RAG | MVP 不做，Phase 5 之后再说 | [D3](../DECISIONS.md) | ✅ 已确认 |
| 7 | 推理后端选什么 | **llama.android JNI**（[D7 修订](../DECISIONS.md)，切回 JNI，废弃 llama-server HTTP） | [D7](../DECISIONS.md)、[X7](../DECISIONS.md) | ✅ 已确认 |
| 8 | 模型存储路径 | **DE 加密区 + Direct Boot**（[D21 新增](../DECISIONS.md)） | [D21](../DECISIONS.md) | ✅ 已确认 |
| 9 | ASR/TTS 模型许可冲突 | 换 Apache 2.0 许可的 sherpa-onnx 官方模型（[D22 新增](../DECISIONS.md)） | [D22](../DECISIONS.md) | ✅ 已确认 |
| 10 | 唤醒词引擎选什么 | sherpa-onnx KWS（[D23 新增](../DECISIONS.md)，弃用 openWakeWord） | [D23](../DECISIONS.md) | ✅ 已确认 |
| - | Phantom Mic 上游活跃度 | 待 Phase 1 启动前复查（v2.0 已近 2 年未更新） | [D14](../DECISIONS.md) | 🟡 待复查 |
| - | 模型 SHA256 校验值 | 待 Phase 1 release 时填 | [D13](../DECISIONS.md) | 🟡 待 Phase 1 |
| - | 中文唤醒词样本 | Phase 3 时采集 100+ 句"嘿小脑"样本 | [D15](../DECISIONS.md) | 🟡 待 Phase 3 |

---

## 14. 当前阶段的明确边界

✅ **本阶段已交付**（Phase 0 设计冻结，第二轮深度检查后修订）：
- 完整设计文档 **14 份**（[docs/README.md](./README.md)）：含本文 + 02-13 共 14 份
  - 00-08 核心设计 9 份
  - 09-13 扩展设计稿 5 份（Phase 6-10）
- 决策清单 [DECISIONS.md](../DECISIONS.md)（**31 条已确认（D1-D31） + 7 条已废弃（X1-X7）**，共 38 条，第四轮 web 核实后修订）
- 借鉴清单（本文 §3，含完整仓库链接）+ 风险评估（本文 §10）+ 路线图（本文 §11）
- Issue 模板 3 份（bug / feature / test_report）
- 第三方许可声明 [third_party/NOTICES.md](../third_party/NOTICES.md)
- 三个目录骨架（[app/](../app/) / [ksu_module/](../ksu_module/) / [scripts/](../scripts/)）的 README 占位

❌ **本阶段未交付**（按用户要求"先不要交付"）：
- 任何 .kt / .cpp / .sh / .gradle 代码
- 任何二进制（libllama.so、libsherpa-onnx-jni.so、模型）
- 任何可执行物
- KSU 模块 zip
- APK
- GitHub Actions workflows（Phase 5 才补）

下一步等用户确认进入 Phase 1 后开始编码。
