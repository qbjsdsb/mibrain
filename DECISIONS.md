# MiBrain 开放决策清单

> 此文件记录所有项目设计决策。  
> ✅ 已确认 | 🟡 待讨论 | ❌ 已废弃

## 已确认决策

### ✅ D1. 默认对话模型
- **选择**: Qwen2.5-3B-Instruct GGUF Q4_K_M (~2GB)
- **理由**: 8GB 内存能扛住，质量明显比 1.5B 好
- **日期**: 2026-06-30
- **状态**: 已确认
- **备选**: Qwen2.5-1.5B-Instruct Q4_K_M（如内存压力大，可切换）

### ✅ D2. 唤醒词策略
- **选择**: MVP 用英文 `hey_jarvis`，Phase 3 再训中文唤醒词
- **理由**: openWakeWord 没有现成中文模型，自训需 1-2 天，不应阻塞 MVP
- **日期**: 2026-06-30
- **状态**: 已确认
- **备选**: 桌面快捷方式手动触发（如唤醒词检测不稳，降级）

### ✅ D3. 是否做 RAG
- **选择**: MVP 不做，Phase 5 之后再说
- **理由**: ToolNeuron 已有完整 RAG 实现，用户可装它补这个能力；MiBrain 专注语音对话核心
- **日期**: 2026-06-30
- **状态**: 已确认

### ✅ D4. 项目名
- **选择**: MiBrain
- **理由**: 与 HyperOS 对应，有辨识度
- **日期**: 2026-06-30
- **状态**: 已确认

### ✅ D5. Root 环境
- **选择**: KernelSU + ZygiskNext + LSPosed
- **理由**: 用户硬约束
- **日期**: 2026-06-30
- **状态**: 已确认

### ✅ D6. 设备
- **选择**: 红米 K50 Ultra（骁龙 8+ Gen 1, 8GB RAM, HyperOS）
- **理由**: 用户硬件
- **日期**: 2026-06-30
- **状态**: 已确认

### ✅ D7. 推理后端
- **选择**: llama.cpp b9844 + llama-server 二进制（不再用 JNI 集成）
- **理由**: 见 docs/02_second_review.md 发现 1。HTTP 调用比 JNI 简单 70%
- **日期**: 2026-06-30
- **状态**: 已确认
- **推翻**: 之前考虑过 fork ToolNeuron 的 LlamaEngine.kt，被验证不可行（实际是 C++ + JNI）

### ✅ D8. ASR/TTS 引擎
- **选择**: sherpa-onnx v1.13.3 AAR（官方 Kotlin API）
- **理由**: 已验证 AAR 含 arm64-v8a 的 .so，官方维护
- **日期**: 2026-06-30
- **状态**: 已确认

### ✅ D9. LSPosed 方案
- **选择**: 装现成的 Phantom Mic v2.0，不自写 hook
- **理由**: 已验证是 LSPosed 官方仓库，hook native 层
- **日期**: 2026-06-30
- **状态**: 已确认
- **风险**: HyperOS 2 (Android 15) 兼容性未验证，Phase 4 真机测

### ✅ D10. APK 开发语言
- **选择**: Kotlin + Jetpack Compose
- **理由**: 现代 Android 标准，与 sherpa-onnx Kotlin API 一致
- **日期**: 2026-06-30
- **状态**: 已确认

### ✅ D11. Daemon 选型
- **选择**: 不用 daemon，shell + 现成二进制
- **理由**: 不用 Go（CGO 麻烦），不用 Python（依赖 Termux）
- **日期**: 2026-06-30
- **状态**: 已确认

### ✅ D12. 仓库归属
- **选择**: GitHub 用户名 `qbjsdsb`，仓库名 `mibrain`
- **理由**: 用户已确认
- **日期**: 2026-06-30
- **状态**: 已确认
- **URL**: https://github.com/qbjsdsb/mibrain

## 待讨论项

### 🟡 D13. 模型 SHA256 校验值
- **说明**: docs/05_deploy_guide.md 提到要校验 SHA256，但具体值需 Phase 1 release 时填
- **状态**: 待 Phase 1

### 🟡 D14. Phantom Mic 在 HyperOS 2 上的兼容性
- **说明**: 真机未验证
- **状态**: 待 Phase 4 真机测

### 🟡 D15. 中文唤醒词样本
- **说明**: Phase 3 时需采集 100+ 句"嘿小脑"样本
- **状态**: 待 Phase 3

## 已废弃决策

### ❌ X1. Go daemon
- **说明**: 之前考虑过 Go daemon + NDK 交叉编译
- **废弃原因**: Android 上 `CGO_ENABLED=0` DNS 会崩，NDK 编译门槛高
- **废弃日期**: 2026-06-30

### ❌ X2. fork ToolNeuron LlamaEngine.kt
- **说明**: 之前考虑 fork ToolNeuron 的 JNI 代码省工作量
- **废弃原因**: ToolNeuron 用的是自己写的 C++ + JNI，不是纯 Kotlin
- **废弃日期**: 2026-06-30

### ❌ X3. HostAI + LiteRT-LM
- **说明**: 之前考虑用 HostAI APK 作为推理后端
- **废弃原因**: LiteRT-LM 模型选择面窄，llama.cpp + GGUF 更灵活
- **废弃日期**: 2026-06-30

### ❌ X4. Tasker + AutoVoice 编排
- **说明**: 之前考虑用 Tasker + AutoVoice 做语音对话
- **废弃原因**: Tasker 在国内依赖 Google 服务，且不支持 SSE 流式
- **废弃日期**: 2026-06-30

### ❌ X5. 自写 LSPosed hook
- **说明**: 之前考虑自己写 LSPosed 双 scope 模块
- **废弃原因**: Phantom Mic 已完整覆盖 native hook，无需重造
- **废弃日期**: 2026-06-30
