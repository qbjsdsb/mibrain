# MiBrain

> 针对小米 HyperOS 的本地 LLM 语音助手开箱即用方案  
> 把"模型服务 + 语音 IO + 后台保活"三件事打包到一个 KSU 模块里，刷入即用。

[![Status](https://img.shields.io/badge/status-design%20frozen-brightgreen)](./docs/00_design_overview.md)
[![License](https://img.shields.io/github/license/qbjsdsb/mibrain)](./LICENSE)
[![Platform](https://img.shields.io/badge/platform-Android%20arm64--v8a%20%7C%20HyperOS-blue)](#)
[![Root](https://img.shields.io/badge/root-KernelSU%20%2B%20ZygiskNext%20%2B%20LSPosed-red)](#)
[![Phase](https://img.shields.io/badge/phase-0%20design%20frozen-yellow)](./DECISIONS.md)

## 这是什么

MiBrain 是一个**完全离线、隐私不出手机**的语音助手项目，目标设备为红米 K50 Ultra（骁龙 8+ Gen 1, 8GB RAM, HyperOS）。

它解决的核心痛点：
- 商业助手在国内受限于云识别 + 隐私顾虑
- 现有本地 LLM 方案要么太重（ToolNeuron），要么太碎（HostAI + Tasker + Phantom Mic 自行拼装）
- MIUI/HyperOS 的后台录音限制让通用 APK 锁屏就哑

一句话定位：
> 把"模型服务 + 语音 IO + 后台保活"三件事打包到一个 KSU 模块里，刷入即用。

## 当前状态：设计冻结

所有设计决策已确认（详见 [DECISIONS.md](./DECISIONS.md)）。**未交付任何功能代码**。

设计文档完整，已通过 3 轮严谨审视：

| 轮次 | 主题 | 文档 |
|---|---|---|
| 第 1 轮 | 完整设计 + 验证所有依赖可达 | [00_design_overview.md](./docs/00_design_overview.md), [01_feasibility_verification.md](./docs/01_feasibility_verification.md) |
| 第 2 轮 | 5 个新发现 + 修订（推翻 JNI 集成、改用 HTTP） | [02_second_review.md](./docs/02_second_review.md) |
| 第 3 轮 | 补 6 个遗漏环节（协议、状态机、回环防护、启动顺序等） | [03_architecture_detail.md](./docs/03_architecture_detail.md) |

部署与运维文档也已就绪：
- [05_deploy_guide.md](./docs/05_deploy_guide.md) — 部署指南（含 8 步 MIUI 白名单）
- [06_lspoded_setup.md](./docs/06_lspoded_setup.md) — LSPosed 配置（Phantom Mic 详解）
- [07_troubleshooting.md](./docs/07_troubleshooting.md) — 故障排查（7 大类故障）

## 技术栈

| 组件 | 选型 | 来源 |
|---|---|---|
| LLM 推理 | llama.cpp b9844 (arm64) + llama-server 二进制 | https://github.com/ggml-org/llama.cpp |
| 语音 ASR/TTS/VAD/KWS | sherpa-onnx v1.13.3 AAR | https://github.com/k2-fsa/sherpa-onnx |
| 默认对话模型 | Qwen2.5-3B-Instruct GGUF Q4_K_M | https://huggingface.co/Qwen/Qwen2.5-3B-Instruct-GGUF |
| 唤醒词（MVP） | openWakeWord hey_jarvis | https://huggingface.co/datasets/cscripka/openWakeWord |
| LSPosed 后台录音 hook | Phantom Mic 2.0 | https://github.com/Xposed-Modules-Repo/tn.amin.phantom_mic |
| Root 框架 | KernelSU + ZygiskNext + LSPosed | https://kernelsu.org |
| APK 开发语言 | Kotlin + Jetpack Compose | - |

## 借鉴的开源项目

本项目借鉴以下开源项目（详细借鉴点见 [docs/00_design_overview.md](./docs/00_design_overview.md) §3）：

- **ToolNeuron** (Siddhesh2377/ToolNeuron) — Kotlin APK 项目骨架参考
- **WhatsMicFix-LSPosed** (D4vRAM369) — LSPosed 双 scope 注入思路
- **HostAI** (wannaphong/android-hostai) — LiteRT-LM 在 Android 上的可行证据
- **tasker-mcp** (cj-elevate) — Go 在 Android 部署模式（参考后已废弃此路）

第三方许可详见 [third_party/NOTICES.md](./third_party/NOTICES.md)。

## 路线图

- ✅ Phase 0：设计冻结（当前）
- ⏳ Phase 1：MVP 单链路打通（APK 骨架 + llama-server + 静态对话）
- ⏳ Phase 2：语音链路（sherpa-onnx ASR/TTS + 前台服务）
- ⏳ Phase 3：唤醒词（hey_jarvis → 自训中文）
- ⏳ Phase 4：稳定性（Phantom Mic + appops + 内存监控 + 真机测试）
- ⏳ Phase 5：发布（CI/CD + 性能基准）

详见 [docs/00_design_overview.md](./docs/00_design_overview.md) §11。

## 关键设计决策（全部已确认）

| 决策 | 选择 | 详见 |
|---|---|---|
| 默认模型 | Qwen2.5-3B Q4_K_M | [DECISIONS.md D1](./DECISIONS.md) |
| 唤醒词（MVP） | 英文 hey_jarvis | [DECISIONS.md D2](./DECISIONS.md) |
| 是否做 RAG | 不做 | [DECISIONS.md D3](./DECISIONS.md) |
| 项目名 | MiBrain | [DECISIONS.md D4](./DECISIONS.md) |
| 推理后端 | llama-server 二进制（HTTP 调用，跳过 JNI） | [DECISIONS.md D7](./DECISIONS.md) |
| LSPosed | 装现成 Phantom Mic，不自写 | [DECISIONS.md D9](./DECISIONS.md) |

## 仓库结构

```
mibrain/
├── README.md                    项目介绍（本文档）
├── LICENSE                       Apache 2.0
├── CONTRIBUTING.md              贡献指南
├── DECISIONS.md                  全部决策清单
├── .gitignore
├── docs/                         设计文档（8 份）
│   ├── 00_design_overview.md    完整设计
│   ├── 01_feasibility_verification.md  验证报告
│   ├── 02_second_review.md      第二轮修订
│   ├── 03_architecture_detail.md 详细架构
│   ├── 05_deploy_guide.md       部署指南
│   ├── 06_lspoded_setup.md      LSPosed 配置
│   └── 07_troubleshooting.md    故障排查
├── app/                          APK 项目骨架（Phase 1 后填代码）
├── ksu_module/                   KernelSU 模块骨架
├── scripts/                      辅助脚本骨架
├── third_party/                  第三方组件许可声明
└── .github/ISSUE_TEMPLATE/       Issue 模板
```

## 许可证

Apache License 2.0 — 见 [LICENSE](./LICENSE)
