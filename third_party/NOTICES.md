# 第三方组件许可声明

> 本项目使用以下开源组件，按各自许可协议使用。

> **2026-06-30 修订说明**：
> - **D7 修订**：llama.cpp 用途从 "作为 llama-server 二进制分发" 改为 "通过 JNI 调用（自编译 libllama.so + libggml.so 打入 APK `jniLibs/arm64-v8a/`）"
> - **D1 修订**：默认对话模型从 Qwen2.5-3B-Instruct 改为 Qwen2.5-1.5B-Instruct（3B 在 8GB 设备必 OOM（[D30](../DECISIONS.md)），仅 12GB+ 设备或 Phase 11+ Adreno GPU 加速后可选）
> - **D22 新增**：ASR/TTS 模型换 Apache 2.0 许可的 sherpa-onnx 官方模型（弃用 CC BY-NC 4.0 的 paraformer + CC BY-NC-ND 4.0 的 aishell3）
> - **D23 新增**：唤醒词改用 sherpa-onnx KWS（弃用 openWakeWord，本文件移除 openWakeWord 条目）
> - **X2 修正**：ToolNeuron 重新评估为 Kotlin + Compose 全栈，作为参考样板
> - **新增**：llama.android 官方 JNI 模块（参考实现来源）
> - 详见 [DECISIONS.md](../DECISIONS.md)

---

## 推理引擎

### llama.cpp
- 仓库：https://github.com/ggml-org/llama.cpp
- 版本：b9830（第四轮 [F3](../docs/14_feasibility_recheck_and_plan.md) 订正：原 b9844 在 2026-06-30 尚未发布，b9830 为 2026-06-28 最新 release）
- 许可：MIT License
- 用途：本地 LLM 推理（GGUF 格式）。通过自编译 `libllama.so` + `libggml.so` 打入 APK 的 `jniLibs/arm64-v8a/`，由 Kotlin 通过 JNI 调用（[D7](../DECISIONS.md)）
- 许可文本：https://github.com/ggml-org/llama.cpp/raw/master/LICENSE

### llama.android（官方 JNI 模块）
- 仓库：https://github.com/ggml-org/llama.cpp/tree/master/examples/llama.android
- 许可：MIT License（随 llama.cpp 主仓库）
- 用途：llama.cpp 官方 Android JNI 集成模块，作为本项目 JNI wrapper 的实现基线（[D7](../DECISIONS.md)）
- 许可文本：https://github.com/ggml-org/llama.cpp/raw/master/LICENSE

### sherpa-onnx
- 仓库：https://github.com/k2-fsa/sherpa-onnx
- 版本：v1.13.3
- 许可：Apache License 2.0
- 用途：全栈语音能力——流式 ASR、TTS、VAD、关键词检测（KWS，[D23](../DECISIONS.md)）。单一 AAR 同时提供 `libsherpa-onnx-jni.so` + `libonnxruntime.so` 在 `jniLibs/arm64-v8a/`
- 许可文本：https://github.com/k2-fsa/sherpa-onnx/raw/master/LICENSE

### ONNX Runtime
- 仓库：https://github.com/microsoft/onnxruntime
- 许可：MIT License
- 用途：sherpa-onnx 推理后端（随 sherpa-onnx AAR 分发）
- 许可文本：https://github.com/microsoft/onnxruntime/raw/main/LICENSE

---

> **许可文本存放策略**：当前阶段未在 `third_party/licenses/` 下保存完整许可文本副本，所有许可文本请访问上述上游仓库链接获取。Phase 1 进入正式分发（APK / KSU 模块 zip）前，会按各自许可证要求补充完整文本到 `third_party/licenses/`。

## 模型

### Qwen2.5-1.5B-Instruct GGUF（默认对话模型）
- 仓库：https://huggingface.co/Qwen/Qwen2.5-1.5B-Instruct-GGUF
- 许可：Qwen License（详见原仓库，允许商用，与项目 Apache 2.0 兼容）
- 用途：默认对话模型（[D1](../DECISIONS.md)）
- 量化：Q4_K_M（~1GB）

### Qwen2.5-3B-Instruct GGUF（备选对话模型）
- 仓库：https://huggingface.co/Qwen/Qwen2.5-3B-Instruct-GGUF
- 许可：Qwen License（详见原仓库）
- 用途：⚠️ 8GB 设备必 OOM 不可用，仅 12GB+ 设备或 Phase 11+ Adreno GPU 加速后可选（[D30](../DECISIONS.md)；原 [D1](../DECISIONS.md) "质量优先可选"已被推翻）
- 量化：Q4_K_M（~2GB）

### sherpa-onnx streaming-zipformer-bilingual-zh-en（流式 ASR）
- 仓库：https://github.com/k2-fsa/sherpa-onnx/releases (asr-models)
- 许可：Apache License 2.0
- 用途：中英双语流式 ASR（[D22](../DECISIONS.md)）

### sherpa-onnx vits-zh-ll（中文 TTS）
- 仓库：https://huggingface.co/k2-fsa/sherpa-onnx
- 许可：社区贡献，许可未明确声明（HF 卡 metadata 缺失）。模型卡仅注明"This model is contributed by the community and trained using https://github.com/Plachtaa/VITS-fast-fine-tuning"，**官方未声明许可**，训练数据来源不明，**不可声称 Apache 2.0**
- 用途：中文 TTS（[D22](../DECISIONS.md)，替代原 aishell3）
- 风险与替代：分发 APK + 此模型可能存在许可风险；~~推荐升级到 `matcha-icefall-zh-baker`（Apache 2.0 已验证）~~ **第四轮 [F7](../docs/14_feasibility_recheck_and_plan.md) 订正：matcha-icefall-zh-baker 训练数据来自 Data-Baker，受 NC（非商业）限制，同样不可分发**。Phase 5 发布前必须另寻 Apache 2.0 中文 TTS 或自训，详见 [D22](../DECISIONS.md)

### silero_vad（VAD）
- 仓库：https://github.com/k2-fsa/sherpa-onnx/releases
- 许可：Apache License 2.0（sherpa-onnx release 自带）
- 用途：语音活动检测

### sherpa-onnx kws-zipformer-wenetspeech-3.3M-2024-01-01（唤醒词 KWS）
- 仓库：https://github.com/k2-fsa/sherpa-onnx/releases (kws-models)
- 许可：Apache License 2.0
- 用途：唤醒词检测（[D23](../DECISIONS.md)）
- 说明：原引用 `zh-vgg` 不存在，真实模型为 zipformer 架构（sherpa-onnx 官方 KWS 预训练清单全部为 Zipformer 架构）

### ~~sherpa-onnx-streaming-paraformer-bilingual-zh-en~~（已弃用）
- 原许可：CC BY-NC 4.0
- 弃用原因：与项目 Apache 2.0 许可冲突（[D22](../DECISIONS.md)）
- 替代：上述 streaming-zipformer-bilingual-zh-en

### ~~vits-icefall-zh-aishell3~~（已弃用）
- 原许可：CC BY-NC-ND 4.0
- 弃用原因：与项目 Apache 2.0 许可冲突（[D22](../DECISIONS.md)）
- 替代：上述 vits-zh-ll

### ~~openWakeWord hey_jarvis~~（已弃用）
- 仓库：https://github.com/dscripka/openWakeWord
- 原许可：MIT License
- 弃用原因：改用 sherpa-onnx 自带 KWS 模块统一技术栈（[D23](../DECISIONS.md)）

## 系统 / LSPosed

### Phantom Mic
- 仓库：https://github.com/Xposed-Modules-Repo/tn.amin.phantom_mic
- 许可：详见原仓库
- 用途：hook AudioRecord.cpp 解决 MIUI/HyperOS 3 后台录音
- 说明：用户自行安装，不打包进本项目
- 风险：v2.0 自 2024-07 发布至本次设计冻结（2026-06-30）已近 2 年未更新，**第四轮 web 核实确认上游已停滞 23 个月**（自 2024-07-24 v2.0 后零更新），HyperOS 3（Android 15）兼容性未验证且无官方更新（[D14](../DECISIONS.md)）

### KernelSU
- 仓库：https://github.com/tiann/KernelSU
- 许可：GNU General Public License v3.0
- 用途：root 框架
- 说明：用户自行安装

### ZygiskNext
- 仓库：https://github.com/Dr-TSNG/ZygiskNext
- 许可：**专有许可（all rights reserved，自 v4-0.9.2 起变更；早期版本为 GPL-3.0）**
- 用途：提供 Zygisk 运行时给 LSPosed
- 说明：用户自行安装。⚠️ 第六轮 R6 核实：自 v4-0.9.2 起许可证从 GPL-3.0 改为 "all rights reserved"，**禁止修改/再分发/提取模块内文件**。本项目仅引导用户自行安装，不分发 ZygiskNext 本体，合规；但若未来需在 KSU 模块内打包或修改 ZygiskNext，须另获作者授权

### LSPosed（Vector fork）
- 仓库：https://github.com/JingMatrix/Vector
- 许可：GNU General Public License v3.0
- 用途：Xposed 框架，加载 Phantom Mic
- 说明：用户自行安装。原 `LSPosed/LSPosed` 仓库已 archive（停止维护），本项目按 [D9](../DECISIONS.md) 使用社区 fork `JingMatrix/Vector`（详见 [D9](../DECISIONS.md)）

## 借鉴项目（不直接使用代码，仅借鉴架构思路）

### ToolNeuron
- 仓库：https://github.com/Siddhesh2377/ToolNeuron
- 许可：MIT License
- 用途：参考 Kotlin + Compose + llama.cpp + sherpa-onnx 全栈同构项目的工程实践，特别是 `InferenceService.kt` + `InferenceClient.kt`（位于 `service/inference/` 目录）JNI 调用范式、AES-256-GCM 模型加密存储、RAG 集成（[X2 重新评估](../DECISIONS.md)）

### WhatsMicFix-LSPosed
- 仓库：https://github.com/D4vRAM369/WhatsMicFix-LSPosed
- 许可：详见原仓库
- 用途：参考 LSPosed 双 scope 注入架构

### HostAI（已弃用参考）
- 仓库：https://github.com/wannaphong/android-hostai
- 许可：详见原仓库
- 用途：原参考 LiteRT-LM 在 Android 上的部署模式（[X3](../DECISIONS.md) 弃用此路）

## Android 框架

### Jetpack Compose / AndroidX
- 许可：Apache License 2.0
- 用途：UI 框架

### Kotlin Coroutines
- 许可：Apache License 2.0
- 用途：异步编程

### Room
- 许可：Apache License 2.0
- 用途：本地数据库

---

如需获取上述组件的完整许可文本，请访问对应仓库。
