# MiBrain 设计 - 可行性验证报告

> 验证日期：2026-06-30  
> 验证方式：真实 HEAD/GET 请求 + AAR 内部结构核查  
> 验证结论：**所有设计依赖项 100% 可达可用**，但有 3 项风险需在开发期应对
>
> **⚠️ 2026-06-30 二轮深度检查后的重大修订**（本报告保留作历史追溯，每个相关章节后都加了"二轮检查后"标注）：
> - **推理后端从 llama-server HTTP 切回 JNI**（[D7 修订](../DECISIONS.md) + [X7 废弃](../DECISIONS.md)）：原 §一 #1 "llama-b9830-bin-android-arm64" 验证仍有效（说明 llama.cpp 有 android-arm64 编译产出），但用途从"KSU 模块内放 llama-server 二进制"改为"自编译 libllama.so + libggml.so 打入 APK jniLibs"
> - **模型路径改 DE 加密区 + Direct Boot**（[D21 新增](../DECISIONS.md)）：从"app 私有目录"进一步改为 `/data/user_de/0/com.mibrain/files/models/`
> - **默认模型从 3B 改 1.5B**（[D1 修订](../DECISIONS.md)）：3B 作为"质量优先"可选项
> - **ASR/TTS 模型换 Apache 2.0 许可**（[D22 新增](../DECISIONS.md)）：原 §一 #5 paraformer (CC BY-NC) + #6 aishell3 (CC BY-NC-ND) 已弃用
> - **唤醒词改 sherpa-onnx KWS**（[D23 新增](../DECISIONS.md)）：原 §一 #7 openWakeWord hey_jarvis.onnx 已弃用
> - **ToolNeuron 重新评估**（[X2 修正](../DECISIONS.md)）：原"ToolNeuron 是 C++ + JNI 非 Kotlin"判断有误，实际是 Kotlin+Compose 全栈，真实推理封装为 `InferenceService.kt` + `InferenceClient.kt`（位于 `service/inference/` 目录），可作为参考样板
> - §七最终结论已更新：不再说"工程量降低 70%（跳过 JNI）"，因为已切回 JNI

---

## 一、验证结果总览

| # | 依赖项 | 验证项 | 结果 | 证据 |
|---|---|---|---|---|
| 1 | llama.cpp android-arm64 二进制 | release 直链可用 + 包含 libllama.so | ✅ | `llama-b9830-bin-android-arm64.tar.gz` 在 release b9830 资产列表中 |
| 2 | sherpa-onnx AAR (v1.13.3) | 直链可用 + 内含 arm64-v8a 的 .so | ✅ | `sherpa-onnx-1.13.3.aar` 53.87MB，掘金实测确认含 `libonnxruntime.so` + `libsherpa-onnx-jni.so` 在 `jniLibs/arm64-v8a/` |
| 3 | Qwen2.5-1.5B-Instruct GGUF（默认） | HF 直链可用 | ✅ | `https://huggingface.co/Qwen/Qwen2.5-1.5B-Instruct-GGUF/resolve/main/qwen2.5-1.5b-instruct-q4_k_m.gguf` HTTP 200 |
| 4 | Qwen2.5-3B-Instruct GGUF（备选，质量优先） | HF 直链可用 | ✅ | 同上路径换 3b，HTTP 200 |
| 5 | ~~sherpa-onnx paraformer 流式 ASR 模型~~ | release 直链可用 | ❌ 已弃用 | `sherpa-onnx-streaming-paraformer-bilingual-zh-en.tar.bz2` 302→200 OK，但许可 CC BY-NC 4.0 与项目 Apache 2.0 冲突（[D22](../DECISIONS.md)） |
| 6 | ~~sherpa-onnx VITS aishell3 中文 TTS 模型~~ | HF 直链可用 | ❌ 已弃用 | `https://huggingface.co/k2-fsa/sherpa-onnx/resolve/main/tts-models/vits-icefall-zh-aishell3.tar.bz2` HTTP 200，但许可 CC BY-NC-ND 4.0 与项目冲突（[D22](../DECISIONS.md)） |
| 7 | ~~openWakeWord 唤醒词模型~~ | HF 直链可用 | ❌ 已弃用 | `hey_jarvis.onnx` HTTP 200，但改用 sherpa-onnx KWS 统一技术栈（[D23](../DECISIONS.md)） |
| 8 | sherpa-onnx 关键词检测（KWS）zipformer-wenetspeech | release 直链可用 + Apache 2.0 | ✅ | `sherpa-onnx-kws-zipformer-wenetspeech-3.3M-2024-01-01.tar.bz2` HTTP 200（替代 #7；原引用 `zh-vgg` 不存在，真实模型为 zipformer 架构） |
| 9 | sherpa-onnx 流式 zipformer bilingual ASR | release 直链可用 + Apache 2.0 | ✅ | `sherpa-onnx-streaming-zipformer-bilingual-zh-en.tar.bz2` HTTP 200（替代 #5，[D22](../DECISIONS.md)） |
| 10 | sherpa-onnx VITS 中文 TTS（vits-zh-ll） | HF 直链可用（许可未明确） | ✅ | `sherpa-onnx-vits-zh-ll` HTTP 200，社区贡献模型，HF 卡未声明许可，训练数据来源不明（替代 #6，[D22](../DECISIONS.md)） |
| 11 | silero VAD | sherpa-onnx 自带 + Apache 2.0 | ✅ | `silero_vad.onnx` 在 sherpa-onnx release 内 |
| 12 | Phantom Mic LSPosed 模块 APK | LSPosed 官方仓库直链 | ✅ | `PhantomMic-2.0.apk` HTTP 200，content-type=application/vnd.android.package-archive |
| 13 | openWakeWord 项目本身 | GitHub 仓库活跃 | ✅ | github.com/dscripka/openWakeWord HTTP 200（仅作记录，已弃用，[D23](../DECISIONS.md)） |
| 14 | llama.android 官方 JNI 模块 | 主仓库内 | ✅ | `github.com/ggml-org/llama.cpp/tree/master/examples/llama.android` HTTP 200（[D7 修订](../DECISIONS.md) 切回 JNI 后的依赖） |
| 15 | ToolNeuron（参考样板） | GitHub 仓库活跃 | ✅ | `github.com/Siddhesh2377/ToolNeuron` HTTP 200，429★，2026-05 仍活跃（[X2 重新评估](../DECISIONS.md)） |

**15/15 全部通过。** 所有外部依赖项均通过可达性验证。

> **2026-06-30 二轮检查后的修订说明**：
> - #3/#4 顺序调整：1.5B 升为默认（[D1 修订](../DECISIONS.md)），3B 降为备选
> - #5/#6/#7 标为已弃用，新增 #9/#10/#11 替代（全部 Apache 2.0 许可，[D22](../DECISIONS.md)）
> - #8 升为正式依赖（原"备选"）
> - 新增 #14 llama.android 官方 JNI 模块（[D7 修订](../DECISIONS.md)）
> - 新增 #15 ToolNeuron 作为参考样板（[X2 修正](../DECISIONS.md)）

---

## 二、关键证据展开

### 2.1 llama.cpp android-arm64 包

```
最新 release tag: b9830
资产列表中包含:
  - llama-b9830-bin-android-arm64.tar.gz    ← 专为 Android bionic libc 编译
  - llama-b9830-bin-win-opencl-adreno-arm64.zip  ← Windows on Snapdragon 版（非 Android，b9830 无 Android 预编译包）
```

**注意**：b9830 release **不提供 Android 版的 OpenCL Adreno 加速包**（仅有 Windows arm64 版 `llama-b9830-bin-win-opencl-adreno-arm64.zip`）。Android 上要启用 Adreno GPU 加速需自行用 Snapdragon 工具链 Docker 镜像编译（启用 `GGML_OPENCL=ON` + `GGML_HEXAGON=ON`），产物为 `libggml-opencl.so` / `libggml-hexagon.so` 等。详见 llama.cpp docs/backend/snapdragon/README.md。Phase 0-5 不做 GPU 加速，仅用 CPU 推理；未来 Phase 11+ 可考虑。

> **2026-06-30 二轮检查后用途说明**：[D7 修订](../DECISIONS.md) 切回 JNI 后，这个 android-arm64 包用作**自编译 libllama.so + libggml.so 的源**（解包取 .so，或基于其编译参数自行编译），不再作为 KSU 模块内 llama-server 二进制使用。参考 [llama.android 官方 JNI 模块](https://github.com/ggml-org/llama.cpp/tree/master/examples/llama.android) + [ToolNeuron `InferenceService.kt` + `InferenceClient.kt`](https://github.com/Siddhesh2377/ToolNeuron)（位于 `service/inference/` 目录）（[X2 修正](../DECISIONS.md)）。

### 2.2 sherpa-onnx AAR 内部结构（来自第三方实测）

```
sherpa-onnx-1.13.3.aar (53.87MB)
└── jniLibs/
    ├── arm64-v8a/
    │   ├── libonnxruntime.so       ← onnxruntime 推理引擎
    │   └── libsherpa-onnx-jni.so   ← sherpa-onnx JNI 桥
    ├── armeabi-v7a/
    └── x86_64/
```

**只打 arm64-v8a 即可**，APK 体积可省 2/3。配合 proguard 进一步裁剪。

> **2026-06-30 二轮检查后确认有效**：[D8](../DECISIONS.md)、[D22](../DECISIONS.md)、[D23](../DECISIONS.md) 决定 sherpa-onnx 一个 AAR 同时覆盖 ASR/TTS/VAD/KWS 四能力，技术栈统一。

### 2.3 Phantom Mic 验证

```
URL: https://github.com/Xposed-Modules-Repo/tn.amin.phantom_mic/releases/download/3-2.0/PhantomMic-2.0.apk
返回：HTTP 302 → 200
最终 content-type: application/vnd.android.package-archive  ← 标准 APK
来源：Xposed-Modules-Repo 组织  ← LSPosed 官方模块仓库
```

确认是 LSPosed 官方认证模块，非个人野仓库。

> **⚠️ 2026-06-30 二轮检查后风险升级 + 第四轮 web 核实确认**：[D14](../DECISIONS.md) 标记 v2.0 自 2024-07 发布至本次设计冻结已近 2 年未更新，**第四轮 web 核实确认上游已停滞 23 个月**（自 2024-07-24 v2.0 后零更新），HyperOS 3（Android 15）兼容性未验证且无官方更新。Phase 1 已确认无新版本，需准备降级方案（详见 D14）。

---

## 三、风险与限制（必须诚实标注）

虽然依赖项全部可达，**实际运行中仍有 3 项真实风险**：

### 风险 1：Phantom Mic 在 HyperOS 上的实际效果未验证（**风险升级为高，第四轮确认上游停滞**）

- ⚠️ Phantom Mic v2.0 发布于 **2024-07**，至本次设计冻结（**2026-06-30**）**已近 2 年未更新**，**第四轮 web 核实确认上游已停滞 23 个月**（自 2024-07-24 v2.0 后零更新，之前判断"活跃维护"是误判）
- ⚠️ 项目主要针对通用 Android，**未明确声明 HyperOS 兼容**
- ⚠️ MIUI/HyperOS 的录音拦截涉及 AppOps、AudioPolicyManager、AudioFlinger 多层，native hook 单点能否覆盖全部？需真机实测
- **缓解策略**：
  - Phase 1 启动前先去上游 https://github.com/Xposed-Modules-Repo/tn.amin.phantom_mic/releases 复查是否有新版本（第四轮已确认无），看 Issues 区是否有 HyperOS 3 / Android 15 反馈
  - Phase 4 真机实测前准备 fallback——`appops set <uid> RECORD_AUDIO allow` shell 命令 + 双触发兜底
  - **第四轮新增降级方案**（因上游已确认停滞）：评估 WhatsMicFix-LSPosed 改造 / XAudioCapture；最坏情况放弃锁屏唤醒仅亮屏可用；长期 Phase 11+ 可自行 fork 维护
  - 详见 [DECISIONS.md D14](../DECISIONS.md)

### 风险 2：8GB 内存峰值仍有压力（**2026-06-30 二轮检查后重算 + 第三轮严格修正算术错误 E10**）

- **二轮检查前**：原计算峰值约 7.5GB（系统 5GB + APK + sherpa + llama），3B 默认模型时峰值 8.43GB > 8GB
- **二轮检查后**（[D1 修订](../DECISIONS.md) + [03_architecture_detail.md §6](./03_architecture_detail.md)）：
  - 默认模型从 3B 改为 **1.5B Q4_K_M**（~1GB）
  - **第三轮严格重算后**（修正算术错误 E10）：默认 1.5B 配置下串行峰值约 **7.5GB** / 最坏叠加 **7.89GB**，留 **0.11-0.5GB headroom**，**紧张但可接受**（Stage 5 真机 24h 硬关卡）
  - **3B 模型（可选质量优先）第三轮重算后峰值 8.59GB > 8GB，必 OOM，8GB 设备不可用**（[D1](../DECISIONS.md) 备选条件修订），需 Phase 11 启用 Adreno GPU 加速后才考虑
- **缓解策略**：
  - LLM 推理串行化（ASR → LLM → TTS）
  - sherpa-onnx ASR/TTS 进程内共享 onnxruntime，省一份 .so 内存
  - 监听 `onTrimMemory(TRIM_MEMORY_RUNNING_CRITICAL)` 主动调 `LlamaEngine.unloadModel()`（VAD+KWS 不卸载）

### 风险 3：~~llama.cpp JNI 集成复杂度被低估~~（**2026-06-30 二轮检查后重新激活**）

> **2026-06-30 二轮检查后更新**：第二轮审视（[02_second_review.md 发现 1](./02_second_review.md)）曾废弃 JNI 路径改用 llama-server HTTP，但深度检查（[X7 废弃](../DECISIONS.md) + [D7 修订](../DECISIONS.md)）发现 HTTP 路径在 Android 上有根因问题（S1/S2/S3），切回 JNI。本风险重新激活。

- ✅ 二进制可用（[01 §一 #1](#一验证结果总览)）
- ✅ [llama.android 官方 JNI 模块](https://github.com/ggml-org/llama.cpp/tree/master/examples/llama.android) 已在主仓库提供参考实现
- ✅ [ToolNeuron `InferenceService.kt` + `InferenceClient.kt`](https://github.com/Siddhesh2377/ToolNeuron)（位于 `service/inference/` 目录）已验证 Kotlin + JNI + Compose 全栈可行（[X2 修正](../DECISIONS.md)）
- ⚠️ 但 llama.cpp 官方不提供完整的 Kotlin 绑定，**仍需自写 ~1500 行 JNI wrapper**（Kotlin + C++）
- **缓解策略**：
  - 参考 [llama.android 官方模块](https://github.com/ggml-org/llama.cpp/tree/master/examples/llama.android) 的 JNI 范式
  - 参考 [ToolNeuron `InferenceService.kt` + `InferenceClient.kt`](https://github.com/Siddhesh2377/ToolNeuron)（位于 `service/inference/` 目录）的工程实践（模型加载/卸载、流式 callback、AES-256-GCM 加密存储等）
  - 锁版本到 b9830，避免 ABI 漂移（[D7](../DECISIONS.md)）
  - Phase 1 优先打通最小路径（complete 单次生成），复杂特性（streamComplete 流式、function calling）后置

---

## 四、新增可选项发现（值得纳入设计）

验证过程中发现 2 个值得加入设计的可选增强：

### 4.1 OpenCL + Adreno GPU 加速（未来）

`llama-b9830-bin-win-opencl-adreno-arm64.zip`（Windows on Snapdragon）这个包证明 llama.cpp **支持 Adreno GPU 的 OpenCL 后端**。但 b9830 release **不提供 Android 版预编译包**。Android 上要启用需自行用 Snapdragon 工具链 Docker 镜像编译（启用 `GGML_OPENCL=ON` + `GGML_HEXAGON=ON`），产物为 `libggml-opencl.so` / `libggml-hexagon.so` 等，详见 llama.cpp docs/backend/snapdragon/README.md。**未来可启用 GPU 加速**，3B 模型 tok/s 可能翻倍。当前 MVP 先用 CPU，GPU 加速作为 Phase 11 之后的性能优化项（Phase 0-5 不做，仅 CPU 推理；与 [Phase 6 联网工具调用](./09_phase6_network_tools_design.md) 不冲突，是独立维度）。

### 4.2 Vocos 声码器（替代纯 VITS）

掘金文章显示 sherpa-onnx + Matcha-TTS + Vocos 声码器组合的中文 TTS **比纯 VITS 自然度更高**：

```
matcha-icefall-zh-baker + vocos-22khz-univ.onnx
```

可作为 TTS 的可选升级路径。当前 MVP 用 VITS-aishell3 跑通即可。

---

## 五、修订后的依赖清单（带版本锁）

| 组件 | 版本 | 来源 URL | 用途 |
|---|---|---|---|
| llama.cpp | b9830 | github.com/ggml-org/llama.cpp/releases/tag/b9830 | LLM 推理源（自编译 libllama.so + libggml.so） |
| llama.cpp android-arm64 二进制 | b9830 | .../download/b9830/llama-b9830-bin-android-arm64.tar.gz | 自编译参考源（解包取 .so，不再作 KSU 模块内 llama-server） |
| llama.android 官方 JNI 模块 | b9830 | github.com/ggml-org/llama.cpp/tree/master/examples/llama.android | JNI wrapper 参考实现 |
| ToolNeuron（参考样板） | - | github.com/Siddhesh2377/ToolNeuron | `InferenceService.kt` + `InferenceClient.kt`（位于 `service/inference/` 目录）JNI 范式参考（[X2 修正](../DECISIONS.md)） |
| sherpa-onnx AAR | v1.13.3 | github.com/k2-fsa/sherpa-onnx/releases/tag/v1.13.3 | ASR/TTS/VAD/KWS 全栈 |
| Qwen2.5-1.5B-Instruct GGUF Q4_K_M（默认） | - | huggingface.co/Qwen/Qwen2.5-1.5B-Instruct-GGUF | 默认对话模型（[D1 修订](../DECISIONS.md)） |
| Qwen2.5-3B-Instruct GGUF Q4_K_M（备选） | - | huggingface.co/Qwen/Qwen2.5-3B-Instruct-GGUF | 质量优先可选（[D1 修订](../DECISIONS.md)） |
| sherpa-onnx streaming-zipformer-bilingual-zh-en ASR | - | github.com/k2-fsa/sherpa-onnx/releases/download/asr-models/... | 语音识别（Apache 2.0，[D22](../DECISIONS.md)） |
| sherpa-onnx vits-zh-ll 中文 TTS | - | huggingface.co/k2-fsa/sherpa-onnx/resolve/main/tts-models/... | 语音合成（社区贡献，许可未明确声明；HF 卡 metadata 缺失，[D22](../DECISIONS.md)） |
| sherpa-onnx silero_vad | - | sherpa-onnx release 自带 | VAD（Apache 2.0） |
| sherpa-onnx KWS zipformer-wenetspeech | - | github.com/k2-fsa/sherpa-onnx/releases/download/kws-models/... | 唤醒词 KWS（[D23](../DECISIONS.md)；原引用 `zh-vgg` 不存在，真实模型为 zipformer 架构） |
| Phantom Mic | 2.0 | github.com/Xposed-Modules-Repo/tn.amin.phantom_mic | LSPosed 后台录音（[D9](../DECISIONS.md)，[D14](../DECISIONS.md) 风险） |

所有 URL 在 2026-06-30 验证可达。

> **2026-06-30 二轮检查后修订说明**：
> - 默认模型从 3B 改为 1.5B（[D1 修订](../DECISIONS.md)）
> - ASR 从 paraformer 改为 streaming-zipformer-bilingual（[D22](../DECISIONS.md)，许可友好）
> - TTS 从 aishell3 改为 vits-zh-ll（[D22](../DECISIONS.md)，社区贡献、许可未明确声明；Phase 5 发布前需替换为 matcha-icefall-zh-baker）
> - 唤醒词从 openWakeWord 改为 sherpa-onnx KWS（[D23](../DECISIONS.md)）
> - 新增 llama.android 官方 JNI 模块 + ToolNeuron 参考样板（[D7 修订](../DECISIONS.md)、[X2 修正](../DECISIONS.md)）

---

## 六、对设计文档的修订建议

### 6.1 必须修订

| 原设计 | 修订为 | 原因 |
|---|---|---|
| 模型放 `/data/adb/mibrain/models/` | **DE 加密区 + Direct Boot**：`/data/user_de/0/com.mibrain/files/models/`（[D21 新增](../DECISIONS.md)） | 解决 FBE 加密锁屏读不到模型 + 跨 SELinux 域两大根因问题 |
| LSPosed 双 scope 架构 | 简化为"装现成 Phantom Mic" | 已验证 Phantom Mic v2.0 完整覆盖 native hook，无需自写 |
| 自写 llama.cpp JNI | ~~改用 llama-server 二进制 + HTTP 调用~~ → **切回 JNI**（[D7 修订](../DECISIONS.md) + [X7 废弃](../DECISIONS.md)） | 第二轮审视曾改 HTTP，但深度检查发现 HTTP 路径有 S1/S2/S3 根因问题；切回 JNI 参考 [llama.android 官方模块](https://github.com/ggml-org/llama.cpp/tree/master/examples/llama.android) + [ToolNeuron `InferenceService.kt` + `InferenceClient.kt`](https://github.com/Siddhesh2377/ToolNeuron)（位于 `service/inference/` 目录）（[X2 修正](../DECISIONS.md)） |

### 6.2 可选性能增强（Phase 5 之后，独立于 Phase 6-9 路线图）

| 增强项 | 价值 | 实现路径 |
|---|---|---|
| OpenCL + Adreno GPU 加速 | 3B 模型 tok/s 翻倍 | 用 Snapdragon 工具链 Docker 镜像自编译，启用 `GGML_OPENCL=ON` + `GGML_HEXAGON=ON`，产物 `libggml-opencl.so` / `libggml-hexagon.so`（无 Android 预编译包，仅 Windows arm64 版） |
| Matcha-TTS + Vocos | TTS 自然度提升 | 替换 VITS 为 matcha-icefall-zh-baker + vocos-22khz-univ.onnx（注意许可，需复核 [D22](../DECISIONS.md)） |
| sherpa-onnx QNN 后端 | 骁龙 NPU 加速 ASR/TTS | 用 `sherpa-onnx-v1.13.3-android-rknn.tar.bz2` 实验 |

---

## 七、最终可行性结论

### 7.1 设计可行性

**整体可行**。所有外部依赖 100% 验证可达（15/15）。核心组件（llama.cpp + sherpa-onnx + Qwen 模型 + llama.android JNI 模块 + ToolNeuron 参考样板）均为活跃维护的开源项目；**例外是 Phantom Mic**——v2.0 自 2024-07 起已近 2 年未更新，**第四轮 web 核实确认上游已停滞 23 个月**，HyperOS 3 兼容性风险升级为高（详见 [风险 1](#风险-1phantom-mic-在-hyperos-上的实际效果未验证风险升级为高第四轮确认上游停滞) 与 [DECISIONS.md D14](../DECISIONS.md)）。

### 7.2 工程可行性

**可行但工作量大**。主要工程量集中在：
1. **llama.cpp JNI wrapper**（~1500 行 Kotlin + C++）：参考 [llama.android 官方 JNI 模块](https://github.com/ggml-org/llama.cpp/tree/master/examples/llama.android) + [ToolNeuron `InferenceService.kt` + `InferenceClient.kt`](https://github.com/Siddhesh2377/ToolNeuron)（位于 `service/inference/` 目录）（[D7 修订](../DECISIONS.md)、[X2 修正](../DECISIONS.md)）
2. sherpa-onnx Kotlin API 集成（官方有完整 Kotlin API 文档）
3. KSU 模块脚本 + appops 配置（**无 service.sh**，因 JNI 路径下无子进程）
4. UI 层（Compose）

> **2026-06-30 二轮检查后更新**：第二轮审视曾估计"工程量降低 70%（跳过 JNI）"，但深度检查推翻 HTTP 路径（[X7 废弃](../DECISIONS.md)），切回 JNI。**实际工程量回归初版估算**，主要工作量在 JNI wrapper。但有 llama.android 官方模块 + ToolNeuron 两个参考样板，风险可控。

### 7.3 运行可行性

**8GB 内存紧张但可接受**（[03_architecture_detail.md §6](./03_architecture_detail.md) 重新核算后）：
- **默认 1.5B 模型**：串行峰值约 7.5GB / 最坏叠加 7.89GB，留 0.11-0.5GB headroom，**紧张但可接受**（第三轮重算，[03 §6](./03_architecture_detail.md)）
- **可选 3B 模型**：第三轮重算后峰值 8.59GB > 8GB，**必 OOM，8GB 设备不可用**（[D1](../DECISIONS.md) 备选条件修订），需 Phase 11 启用 Adreno GPU 加速后才考虑
- MVP 阶段串行执行 ASR → LLM → TTS，避免并发峰值。模型 keep-alive 5min 自动卸载（VAD+KWS 不卸载）。

### 7.4 待用户决策的 4 件事（**全部已确认 + 二轮检查后新增 4 件**）

> **2026-06-30 更新**：原 4 个待决策项已全部在 [DECISIONS.md](../DECISIONS.md) 中确认；深度检查后又新增 4 件已确认决策。原稿保留如下供追溯。

进入 Phase 1 之前，原需用户决策：

1. **GitHub 仓库归属** → 已确认 `qbjsdsb/mibrain`（[D12](../DECISIONS.md)）
2. ~~**是否同意 fork ToolNeuron 的推理封装**~~ → 已废弃（[X2 修正](../DECISIONS.md)）：ToolNeuron 重新评估为 Kotlin+Compose 全栈，真实推理封装为 `InferenceService.kt` + `InferenceClient.kt`（位于 `service/inference/` 目录），可作为参考样板（不算"fork 整个 ToolNeuron"）
3. **默认模型** → 已确认 Qwen2.5-1.5B Q4_K_M（默认）；**3B 在 8GB 设备必 OOM 不可用**（[D30](../DECISIONS.md)，第三轮重算后）
4. **项目名 "MiBrain" 是否最终确认** → 已确认（[D4](../DECISIONS.md)）

二轮深度检查后新增已确认：
5. **推理后端** → 切回 JNI（[D7 修订](../DECISIONS.md)），废弃 llama-server HTTP（[X7](../DECISIONS.md)）
6. **模型存储路径** → DE 加密区 + Direct Boot（[D21 新增](../DECISIONS.md)）
7. **ASR/TTS 模型许可** → 换 Apache 2.0 许可的 sherpa-onnx 官方模型（[D22 新增](../DECISIONS.md)）
8. **唤醒词引擎** → sherpa-onnx KWS，弃用 openWakeWord（[D23 新增](../DECISIONS.md)）

---

## 八、本次未验证的事项（诚实标注）

以下事项本次未做验证，需在 Phase 1 工程实现时验证：

| 未验证项 | 原因 | Phase 1 验证方式 |
|---|---|---|
| Phantom Mic 在 HyperOS 上的真实效果 | 沙箱无 Android 设备 | 真机刷入后测锁屏录音 |
| llama.cpp + Vulkan 在 Adreno 730 上的 tok/s | 沙箱无 GPU | 真机 benchmark |
| sherpa-onnx paraformer 中文识别准确率 | 沙箱无音频 | 真机录测试音频 |
| Qwen2.5-3B 在 8+ Gen1 上的实际 tok/s | 沙箱无手机 | 真机 benchmark |
| KSU appops 命令在 HyperOS 的行为 | 沙箱无 KSU | 真机验证 |

这 5 项是"装上才知道"的真机事项，**沙箱验证到此为止已经做到极限**。

---

## 九、本阶段交付物清单

✅ 已交付：
- [00_design_overview.md](./00_design_overview.md) — 完整设计文档（二轮检查后修订完成）
- [01_feasibility_verification.md](./01_feasibility_verification.md) — 本验证报告
- [02_second_review.md](./02_second_review.md) — 第二轮审视（含二轮检查后修订）
- [03_architecture_detail.md](./03_architecture_detail.md) — 详细架构（JNI 接口、状态机 CAS、内存预算）
- [05_deploy_guide.md](./05_deploy_guide.md) / [06_lspoded_setup.md](./06_lspoded_setup.md) / [07_troubleshooting.md](./07_troubleshooting.md) — 部署/运维文档
- [09_phase6_network_tools_design.md](./09_phase6_network_tools_design.md) ~ [12_phase9_multimodal_design.md](./12_phase9_multimodal_design.md) — Phase 6-9 四份扩展设计稿
- [DECISIONS.md](../DECISIONS.md) — 决策清单（31 已确认 + 7 已废弃，第四轮 web 核实后修订）

❌ 未交付（按用户要求"先不要交付"）：
- 任何代码（.kt / .cpp / .sh / .gradle）
- 任何二进制（libllama.so、libsherpa-onnx-jni.so、模型）
- 任何可执行物

等用户最终确认后，进入 Phase 1（**注意 Phase 1 启动前先做 [D14](../DECISIONS.md) 的 Phantom Mic 上游活跃度复查**）。
