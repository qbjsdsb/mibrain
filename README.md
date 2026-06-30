# MiBrain

> 针对小米 HyperOS 3 的完全离线本地 LLM 语音助手方案
> 把"模型推理 + 语音 IO + 后台保活"三件事打包到一个 APK + KSU 模块里，刷入即用。

[![Status](https://img.shields.io/badge/status-design%20frozen-brightgreen)](./docs/00_design_overview.md)
[![License](https://img.shields.io/github/license/qbjsdsb/mibrain)](./LICENSE)
[![Platform](https://img.shields.io/badge/platform-Android%20arm64--v8a%20%7C%20HyperOS%203-blue)](#)
[![Root](https://img.shields.io/badge/root-KernelSU%20%2B%20ZygiskNext%20%2B%20LSPosed%20Vector-red)](#)
[![Phase](https://img.shields.io/badge/phase-0%20design%20frozen-yellow)](./DECISIONS.md)

> **2026-06-30 修订说明**（深度检查 + 开源调研后第二轮调整 + 第四轮 web 核实订正）：
> - **D7 修订**：推理后端从 llama-server HTTP 切回 JNI（参考 [llama.android](https://github.com/ggml-org/llama.cpp/tree/master/examples/llama.android) 官方模块 + [ToolNeuron](https://github.com/Siddhesh2377/ToolNeuron) `InferenceService.kt` + `InferenceClient.kt`（位于 `service/inference/` 目录）样板），不再有独立子进程
> - **D7 版本号订正（第四轮 [F3](./docs/14_feasibility_recheck_and_plan.md)）**：llama.cpp 锁定版本从 b9844 改为 **b9830**（b9844 在 2026-06-30 尚未发布，b9830 为 2026-06-28 最新 release）
> - **X7 废弃**：llama-server HTTP 路径废弃，详见 [DECISIONS.md](./DECISIONS.md)
> - **D1 修订**：默认模型从 3B 改为 1.5B Q4_K_M（~1GB），3B 作为质量优先可选（第四轮重算后 3B 在 8GB 设备必 OOM，详见 [03 §6](./docs/03_architecture_detail.md)）
> - **D21 新增**：模型路径改 DE 加密区 + Direct Boot（`/data/user_de/0/com.mibrain/files/models/`）
> - **D22 新增**：ASR/TTS 模型换 Apache 2.0 许可的 sherpa-onnx 官方模型（弃用 CC BY-NC 的 paraformer/aishell3；第四轮 [F7](./docs/14_feasibility_recheck_and_plan.md) 确认 matcha-icefall-zh-baker 也不是合规替代，受 Data-Baker NC 限制）
> - **D23 新增**：唤醒词改用 sherpa-onnx KWS（弃用 openWakeWord，统一 sherpa-onnx 全栈；第四轮 [F6](./docs/14_feasibility_recheck_and_plan.md) 修正 WenetSpeech 许可表述为"自相矛盾"）
> - **D14 风险升级（第四轮 [F5](./docs/14_feasibility_recheck_and_plan.md)）**：Phantom Mic 上游已确认停滞 23 个月，补降级方案（appops + 双触发兜底 / WhatsMicFix 改造 / 放弃锁屏唤醒）
> - **X2 修正**：ToolNeuron 重新评估为 Kotlin + Compose 全栈，作为参考样板而非"非纯 Kotlin 弃用"
> - 文档从 8 份扩展为 12 份编号文档（新增 Phase 6-10 共 5 份设计稿），决策从 6 条扩为 **31 项已确认**（D1-D31） + 7 项已废弃（X1-X7，含 1 项重新评估）

## 这是什么

MiBrain 是一个**完全离线、隐私不出手机**的语音助手项目，目标设备为红米 K50 Ultra（骁龙 8+ Gen 1, 8GB RAM, HyperOS 3）。

它解决的核心痛点：
- 商业助手在国内受限于云识别 + 隐私顾虑
- 现有本地 LLM 方案要么是参考样板（[ToolNeuron](https://github.com/Siddhesh2377/ToolNeuron)）而非开箱即用，要么太碎（HostAI + Tasker + Phantom Mic 自行拼装）
- MIUI/HyperOS 3 的后台录音限制让通用 APK 锁屏就哑

一句话定位：
> 把"模型推理 + 语音 IO + 后台保活"三件事打包到一个 APK + KSU 模块里，刷入即用。

## 当前状态：设计冻结

所有设计决策已确认（详见 [DECISIONS.md](./DECISIONS.md)）。**未交付任何功能代码**。

设计文档完整，已通过 3 轮严谨审视 + 1 轮深度检查 + 二轮修订：

| 轮次 | 主题 | 文档 |
|---|---|---|
| 第 1 轮 | 完整设计 + 验证所有依赖可达 | [00_design_overview.md](./docs/00_design_overview.md), [01_feasibility_verification.md](./docs/01_feasibility_verification.md) |
| 第 2 轮 | 5 个新发现 + 修订 | [02_second_review.md](./docs/02_second_review.md) |
| 第 3 轮 | 补 6 个遗漏环节（协议、状态机、回环防护、启动顺序等） | [03_architecture_detail.md](./docs/03_architecture_detail.md) |
| 深度检查 + 二轮修订 | 推翻 HTTP 改回 JNI、DE 加密区 + Direct Boot、1.5B 默认、sherpa-onnx 全栈统一 | 见各文档顶部修订说明 blockquote |

部署与运维文档也已就绪：
- [05_deploy_guide.md](./docs/05_deploy_guide.md) — 部署指南（含 9 步 MIUI 白名单，HyperOS 2.0+ 含"应用智能休眠"关闭）
- [06_lspoded_setup.md](./docs/06_lspoded_setup.md) — LSPosed 配置（Phantom Mic 详解）
- [07_troubleshooting.md](./docs/07_troubleshooting.md) — 故障排查（7 大类故障）

Phase 6-9 扩展功能设计稿：
- [09_phase6_network_tools_design.md](./docs/09_phase6_network_tools_design.md) — Phase 6 联网工具调用（联网开关 + 4 工具）
- [10_phase7_phone_control_design.md](./docs/10_phase7_phone_control_design.md) — Phase 7 手机控制（4 工具）
- [11_phase8_platform_design.md](./docs/11_phase8_platform_design.md) — Phase 8 平台化（API/Widget/MCP）
- [12_phase9_multimodal_design.md](./docs/12_phase9_multimodal_design.md) — Phase 9 多模态（OCR + 看图对话）

## 技术栈

| 组件 | 选型 | 来源 |
|---|---|---|
| LLM 推理 | llama.cpp b9830 + 官方 [llama.android](https://github.com/ggml-org/llama.cpp/tree/master/examples/llama.android) JNI 模块（参考 [ToolNeuron](https://github.com/Siddhesh2377/ToolNeuron) `InferenceService.kt` + `InferenceClient.kt`） | https://github.com/ggml-org/llama.cpp |
| 语音 ASR/TTS/VAD/KWS（全栈） | sherpa-onnx v1.13.3 AAR | https://github.com/k2-fsa/sherpa-onnx |
| 默认对话模型 | Qwen2.5-1.5B-Instruct GGUF Q4_K_M（~1GB） | https://huggingface.co/Qwen/Qwen2.5-1.5B-Instruct-GGUF |
| 备选对话模型 | Qwen2.5-3B-Instruct GGUF Q4_K_M（~2GB，质量优先可选） | https://huggingface.co/Qwen/Qwen2.5-3B-Instruct-GGUF |
| 流式 ASR | sherpa-onnx streaming-zipformer-bilingual-zh-en（Apache 2.0） | https://github.com/k2-fsa/sherpa-onnx/releases |
| 中文 TTS | sherpa-onnx vits-zh-ll（社区贡献，许可未明确声明；**matcha-icefall-zh-baker 不是合规替代**，受 Data-Baker NC 限制，[D22](./DECISIONS.md)） | https://huggingface.co/k2-fsa/sherpa-onnx |
| VAD | silero_vad（Apache 2.0） | sherpa-onnx release 自带 |
| 唤醒词 | sherpa-onnx KWS zipformer-wenetspeech（Apache 2.0） | https://github.com/k2-fsa/sherpa-onnx/releases |
| 模型存储 | DE 加密区 + Direct Boot（`/data/user_de/0/com.mibrain/files/models/`） | [D21](./DECISIONS.md) |
| LSPosed 后台录音 hook | Phantom Mic 2.0 | https://github.com/Xposed-Modules-Repo/tn.amin.phantom_mic |
| Root 框架 | KernelSU + ZygiskNext + LSPosed Vector（[D9](./DECISIONS.md)，第四轮 [F4](./docs/14_feasibility_recheck_and_plan.md) 订正来源 `JingMatrix/Vector`） | https://kernelsu.org |
| APK 开发语言 | Kotlin + Jetpack Compose | - |
| JNI wrapper | C++ + JNI（参考 ToolNeuron `InferenceService.kt` + `InferenceClient.kt`） | 见 `app/src/main/jni/` |

## 借鉴的开源项目

本项目借鉴以下开源项目（详细借鉴点见 [docs/00_design_overview.md](./docs/00_design_overview.md) §3）：

- **[ToolNeuron](https://github.com/Siddhesh2377/ToolNeuron)** (Siddhesh2377) — Kotlin + Compose + llama.cpp + sherpa-onnx 全栈同构项目，`InferenceService.kt` + `InferenceClient.kt`（位于 `service/inference/` 目录）JNI 调用范式 + AES-256-GCM 模型加密存储 + RAG 工程实践作为参考样板（429★，2026-05 仍活跃）
- **[llama.android](https://github.com/ggml-org/llama.cpp/tree/master/examples/llama.android)** (ggml-org 官方) — llama.cpp 官方 Android JNI 集成模块，作为 JNI wrapper 的实现基线
- **[sherpa-onnx 官方 Android 示例](https://github.com/k2-fsa/sherpa-onnx/tree/master/android)** (k2-fsa) — ASR/TTS/VAD/KWS Kotlin API 调用范式
- **[WhatsMicFix-LSPosed](https://github.com/D4vRAM369/WhatsMicFix-LSPosed)** (D4vRAM369) — LSPosed 双 scope 注入思路
- **[HostAI](https://github.com/wannaphong/android-hostai)** (wannaphong) — LiteRT-LM 在 Android 上的可行证据（参考后已弃用此路）
- **tasker-mcp** (cj-elevate) — Go 在 Android 部署模式（参考后已弃用此路）

第三方许可详见 [third_party/NOTICES.md](./third_party/NOTICES.md)。

## 路线图

- ✅ Phase 0：设计冻结（当前）
- ⏳ Phase 1：MVP 单链路打通（APK 骨架 + JNI wrapper + LlamaEngine 加载 + 静态对话 + Direct Boot 声明）
- Phase 2：语音链路（sherpa-onnx ASR/TTS/VAD + 前台服务）
- Phase 3：唤醒词（sherpa-onnx KWS hey_jarvis → 自训中文）
- Phase 4：稳定性（Phantom Mic + appops + 内存监控 + 真机测试 + D14 复查）
- Phase 5：发布（CI/CD + 性能基准）
- Phase 6：联网工具调用（联网开关 + 天气/翻译/搜索/新闻）— [设计稿](./docs/09_phase6_network_tools_design.md)
- Phase 7：手机控制（通知朗读/应用启动/系统设置/截屏看屏）— [设计稿](./docs/10_phase7_phone_control_design.md)
- Phase 8：插件系统 + API 暴露（APK 内嵌 HTTP server + Widget + MCP）— [设计稿](./docs/11_phase8_platform_design.md)
- Phase 9：多模态（OCR/看图/实时翻译）— [设计稿](./docs/12_phase9_multimodal_design.md)
- Phase 10：多用户 + 儿童模式 + i18n + a11y + 音频路由 + 全局暂停 — [设计稿](./docs/13_phase10_ux_enhancements_design.md)

详见 [docs/00_design_overview.md](./docs/00_design_overview.md) §11。

## 关键设计决策

> 完整决策清单见 [DECISIONS.md](./DECISIONS.md)：31 项已确认（D1-D31） + 7 项已废弃/重新评估（X1-X7）。

| 决策 | 选择 | 详见 |
|---|---|---|
| 默认模型 | Qwen2.5-1.5B Q4_K_M（默认）；**3B 在 8GB 设备必 OOM 不可用**（[D30](./DECISIONS.md)），仅 12GB+ 设备或 Phase 11+ Adreno GPU 加速后可选 | [D1](./DECISIONS.md) + [D30](./DECISIONS.md) |
| 唤醒词（MVP） | 英文 hey_jarvis | [D2](./DECISIONS.md) |
| 是否做 RAG | MVP 不做 | [D3](./DECISIONS.md) |
| 项目名 | MiBrain | [D4](./DECISIONS.md) |
| Root 环境 | KernelSU + ZygiskNext + LSPosed | [D5](./DECISIONS.md) |
| 设备 | 红米 K50 Ultra（骁龙 8+ Gen 1, 8GB RAM, HyperOS 3） | [D6](./DECISIONS.md) |
| 推理后端 | llama.cpp b9830 + 官方 examples/llama.android JNI 模块 | [D7](./DECISIONS.md) |
| ASR/TTS 引擎 | sherpa-onnx v1.13.3 AAR | [D8](./DECISIONS.md) |
| LSPosed | 装现成 Phantom Mic，不自写 | [D9](./DECISIONS.md) |
| APK 开发语言 | Kotlin + Jetpack Compose | [D10](./DECISIONS.md) |
| 联网开关语义 | 全局开关 + 每工具子开关 | [D16](./DECISIONS.md) |
| 工具调用机制 | 关键词正则优先 + LLM function calling 兜底 | [D17](./DECISIONS.md) |
| 多模态视觉 API | 默认 GLM-4V，备选 GPT-4V/Claude/Qwen-VL | [D20](./DECISIONS.md) |
| 模型存储路径 | DE 加密区 + Direct Boot | [D21](./DECISIONS.md) |
| ASR/TTS 模型许可 | Apache 2.0（弃用 CC BY-NC） | [D22](./DECISIONS.md) |
| 唤醒词引擎 | sherpa-onnx KWS（弃用 openWakeWord） | [D23](./DECISIONS.md) |

待讨论 / 风险升级：

| 决策 | 状态 | 详见 |
|---|---|---|
| 模型 SHA256 校验值 | 待 Phase 1 release 时填 | [D13](./DECISIONS.md) |
| Phantom Mic 在 HyperOS 3 兼容性 | **第四轮确认上游已停滞 23 个月**，风险高，已补降级方案 | [D14](./DECISIONS.md) |
| 中文唤醒词样本 | 待 Phase 3 采集 | [D15](./DECISIONS.md) |

已废弃路径（避免重复踩坑）：

| 决策 | 废弃原因 | 详见 |
|---|---|---|
| Go daemon | CGO_ENABLED=0 DNS 崩 + NDK 编译门槛高 | [X1](./DECISIONS.md) |
| HostAI + LiteRT-LM | 模型选择面窄 | [X3](./DECISIONS.md) |
| Tasker + AutoVoice 编排 | 依赖 Google 服务，不支持 SSE | [X4](./DECISIONS.md) |
| 自写 LSPosed hook | Phantom Mic 已完整覆盖 | [X5](./DECISIONS.md) |
| 用户写 Tool.kt 热加载 | 安全风险大，沙箱复杂 | [X6](./DECISIONS.md) |
| llama-server HTTP 路径 | S1/S2/S3 根因性问题（无 Android 先例） | [X7](./DECISIONS.md) |

> 注：[X2](./DECISIONS.md) 原"废弃 fork ToolNeuron"经重新评估**不再废弃**，ToolNeuron 改为参考样板（详见 DECISIONS.md）。

## 仓库结构

```
mibrain/
├── README.md                    项目介绍（本文档）
├── LICENSE                       Apache 2.0
├── CONTRIBUTING.md              贡献指南
├── DECISIONS.md                  全部决策清单（D1-D31 + X1-X7）
├── .gitignore
├── docs/                         设计文档（12 份编号 + 目录索引）
│   ├── README.md                文档目录索引
│   ├── 00_design_overview.md    完整设计
│   ├── 01_feasibility_verification.md  验证报告（15/15 依赖项实测可达）
│   ├── 02_second_review.md      第二轮审视 + 二轮修订
│   ├── 03_architecture_detail.md 详细架构（含状态机 CAS + WakeLock + DE 路径）
│   ├── 05_deploy_guide.md       部署指南
│   ├── 06_lspoded_setup.md      LSPosed 配置
│   ├── 07_troubleshooting.md    故障排查
│   ├── 09_phase6_network_tools_design.md   Phase 6 联网工具调用设计
│   ├── 10_phase7_phone_control_design.md   Phase 7 手机控制设计
│   ├── 11_phase8_platform_design.md       Phase 8 平台化设计
│   ├── 12_phase9_multimodal_design.md     Phase 9 多模态设计
│   └── 13_phase10_ux_enhancements_design.md  Phase 10 UX 增强设计
├── app/                          APK 项目骨架（Phase 1 后填代码）
│   └── src/main/jni/             JNI wrapper（CMakeLists.txt + llama_engine.cpp）
├── ksu_module/                   KernelSU 模块骨架（post-fs-data.sh + appops + sepolicy）
├── scripts/                      辅助脚本骨架（build_ksu_zip.sh + build_llama_jni.sh）
├── third_party/                  第三方组件许可声明
└── .github/ISSUE_TEMPLATE/       Issue 模板
```

> Phase 1 实施后会有 `04_build_guide.md`（编译指南）与 `08_performance_bench.md`（性能基准）两份占位文档补齐。

## 许可证

Apache License 2.0 — 见 [LICENSE](./LICENSE)
