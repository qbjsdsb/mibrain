# 第三方组件许可声明

本项目使用以下开源组件，按各自许可协议使用：

## 推理引擎

### llama.cpp
- 仓库：https://github.com/ggml-org/llama.cpp
- 版本：b9844
- 许可：MIT License
- 用途：本地 LLM 推理（GGUF 格式，作为 llama-server 二进制分发）
- 许可文本：https://github.com/ggml-org/llama.cpp/raw/master/LICENSE

### sherpa-onnx
- 仓库：https://github.com/k2-fsa/sherpa-onnx
- 版本：v1.13.3
- 许可：Apache License 2.0
- 用途：流式 ASR、TTS、VAD、关键词检测
- 许可文本：https://github.com/k2-fsa/sherpa-onnx/raw/master/LICENSE

### ONNX Runtime
- 仓库：https://github.com/microsoft/onnxruntime
- 许可：MIT License
- 用途：sherpa-onnx 推理后端
- 许可文本：https://github.com/microsoft/onnxruntime/raw/main/LICENSE

---

> **许可文本存放策略**：当前阶段未在 `third_party/licenses/` 下保存完整许可文本副本，所有许可文本请访问上述上游仓库链接获取。Phase 1 进入正式分发（APK / KSU 模块 zip）前，会按各自许可证要求补充完整文本到 `third_party/licenses/`。

## 模型

### Qwen2.5-3B-Instruct GGUF
- 仓库：https://huggingface.co/Qwen/Qwen2.5-3B-Instruct-GGUF
- 许可：Qwen License (详见原仓库)
- 用途：默认对话模型

### sherpa-onnx-streaming-paraformer-bilingual-zh-en
- 仓库：https://github.com/k2-fsa/sherpa-onnx/releases (asr-models)
- 许可：CC BY-NC 4.0
- 用途：流式 ASR

### vits-icefall-zh-aishell3
- 仓库：https://huggingface.co/k2-fsa/sherpa-onnx
- 许可：CC BY-NC-ND 4.0
- 用途：中文 TTS

### openWakeWord
- 仓库：https://github.com/dscripka/openWakeWord
- 许可：MIT License
- 用途：唤醒词检测

## 系统 / LSPosed

### Phantom Mic
- 仓库：https://github.com/Xposed-Modules-Repo/tn.amin.phantom_mic
- 许可：详见原仓库
- 用途：hook AudioRecord.cpp 解决 MIUI 后台录音
- 说明：用户自行安装，不打包进本项目

## 借鉴项目（不直接使用代码，仅借鉴架构思路）

### ToolNeuron
- 仓库：https://github.com/Siddhesh2377/ToolNeuron
- 许可：MIT License
- 用途：参考 Kotlin + llama.cpp 集成方案

### WhatsMicFix-LSPosed
- 仓库：https://github.com/D4vRAM369/WhatsMicFix-LSPosed
- 许可：详见原仓库
- 用途：参考 LSPosed 双 scope 注入架构

### HostAI
- 仓库：https://github.com/wannaphong/android-hostai
- 许可：详见原仓库
- 用途：参考 LiteRT-LM 在 Android 上的部署模式

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
