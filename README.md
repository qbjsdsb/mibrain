# MiBrain

> 针对小米 HyperOS 的本地 LLM 语音助手开箱即用方案  
> 把"模型服务 + 语音 IO + 后台保活"三件事打包到一个 KSU 模块里，刷入即用。

[![Status](https://img.shields.io/badge/status-design%20phase-yellow)](./docs/00_design_overview.md)
[![License](https://img.shields.io/github/license/qbjsdsb/mibrain)](./LICENSE)
[![Platform](https://img.shields.io/badge/platform-Android%20arm64--v8a%20%7C%20HyperOS-blue)](#)
[![Root](https://img.shields.io/badge/root-KernelSU%20%2B%20ZygiskNext%20%2B%20LSPosed-red)](#)

## 这是什么

MiBrain 是一个**完全离线、隐私不出手机**的语音助手项目，目标设备为红米 K50 Ultra（骁龙 8+ Gen 1, 8GB RAM, HyperOS）。

它解决的核心痛点：
- 商业助手在国内受限于云识别 + 隐私顾虑
- 现有本地 LLM 方案要么太重（ToolNeuron），要么太碎（HostAI + Tasker + Phantom Mic 自行拼装）
- MIUI/HyperOS 的后台录音限制让通用 APK 锁屏就哑

一句话定位：
> 把"模型服务 + 语音 IO + 后台保活"三件事打包到一个 KSU 模块里，刷入即用。

## 当前状态：设计阶段

仓库目前处于**设计冻结**状态，**尚未交付任何代码**。

已完成的产物（在 `docs/` 目录下）：
- [`00_design_overview.md`](./docs/00_design_overview.md) — 完整设计文档（架构、组件、路线图）
- [`01_feasibility_verification.md`](./docs/01_feasibility_verification.md) — 可行性验证报告（所有依赖项实测可达）
- [`02_second_review.md`](./docs/02_second_review.md) — 第二轮严谨审视（5 个新发现 + 修订）

待回答的开放问题（见 `00_design_overview.md` §13）：
1. 默认模型选 Qwen2.5-3B 还是 1.5B
2. 唤醒词定什么（默认"嘿小脑"，但需自训模型）
3. 是否做 RAG（建议 MVP 不做）
4. Phase 1 何时启动

## 技术栈

| 组件 | 选型 | 来源 |
|---|---|---|
| LLM 推理 | llama.cpp b9844 (arm64) | https://github.com/ggml-org/llama.cpp |
| 语音 ASR/TTS/VAD/KWS | sherpa-onnx v1.13.3 | https://github.com/k2-fsa/sherpa-onnx |
| 默认对话模型 | Qwen2.5-3B-Instruct GGUF Q4_K_M | https://huggingface.co/Qwen/Qwen2.5-3B-Instruct-GGUF |
| LSPosed 后台录音 hook | Phantom Mic 2.0 | https://github.com/Xposed-Modules-Repo/tn.amin.phantom_mic |
| Root 框架 | KernelSU + ZygiskNext + LSPosed | https://kernelsu.org |
| APK 开发语言 | Kotlin + Jetpack Compose | - |

## 借鉴的开源项目

本项目借鉴以下开源项目（详细借鉴点见 `docs/00_design_overview.md` §3）：

- **ToolNeuron** (Siddhesh2377/ToolNeuron) — Kotlin APK 项目骨架
- **WhatsMicFix-LSPosed** (D4vRAM369) — LSPosed 双 scope 架构思路
- **HostAI** (wannaphong/android-hostai) — LiteRT-LM 在 Android 上的可行证据
- **tasker-mcp** (cj-elevate) — Go 在 Android 上的部署模式（参考）

## 路线图

- ✅ Phase 0：设计冻结（当前）
- ⏳ Phase 1：MVP 单链路打通（APK 骨架 + llama.cpp + 静态对话）
- ⏳ Phase 2：语音链路（ASR + TTS + 前台服务）
- ⏳ Phase 3：唤醒词
- ⏳ Phase 4：稳定性（Phantom Mic + appops + 内存监控）
- ⏳ Phase 5：发布

详见 `docs/00_design_overview.md` §11。

## 许可证

Apache License 2.0 — 见 [LICENSE](./LICENSE)

第三方组件许可：见 [`third_party/NOTICES.md`](./third_party/NOTICES.md)
