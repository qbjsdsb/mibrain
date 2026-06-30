# MiBrain 开放决策清单

> 此文件记录所有项目设计决策。  
> ✅ 已确认 | 🟡 待讨论 | ❌ 已废弃

## 已确认决策

### ✅ D1. 默认对话模型（2026-06-30 修订）
- **选择**: **默认 Qwen2.5-1.5B-Instruct GGUF Q4_K_M (~1GB)**，3B 作为"质量优先"可选项
- **修订原因**: 深度检查发现原默认 3B 时内存峰值 8.43GB > 8GB 设备总内存（见 [03_architecture_detail.md §6](./docs/03_architecture_detail.md)），且 VAD + KWS 必须常驻无法释放，"串行执行"假设不成立。降级到 1.5B 后峰值约 6.5-7GB，留 1GB+ headroom
- **首次启动体验**: 1.5B 模型下载体积小（~1GB vs 3B 的 2GB），首次启动更友好
- **状态**: 已确认
- **备选**: Qwen2.5-3B-Instruct Q4_K_M（~2GB，质量优先模式，需用户主动切换 + Phase 4 真机验证不 OOM）
- **修订历史**: 原默认 3B（2026-06-30 设计冻结初版）→ 修订默认 1.5B（同日，深度检查后）

### ✅ D2. 唤醒词策略
- **选择**: MVP 用英文 `hey_jarvis`，Phase 3 再训中文唤醒词
- **理由**: 自训中文唤醒词需 1-2 天采集 + 训练，不应阻塞 MVP
- **日期**: 2026-06-30
- **状态**: 已确认
- **备选**: 桌面快捷方式手动触发（如唤醒词检测不稳，降级）
- **关联**: 引擎从 openWakeWord 改为 sherpa-onnx KWS（[D23](#-d23-唤醒词改用-sherpa-onnx-kws)），原"openWakeWord 没有现成中文模型"的具体理由失效，但 MVP 先英文的结论不变

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

### ✅ D7. 推理后端（2026-06-30 修订：切回 JNI）
- **选择**: **llama.cpp b9844 + 官方 [llama.android](https://github.com/ggml-org/llama.cpp/tree/master/llama.android) JNI 模块**（参考 [ToolNeuron](https://github.com/Siddhesh2377/ToolNeuron) 的 LlamaEngine.kt 实践）
- **修订原因**: 深度检查 + 开源调研后发现：
  1. **无任何先例**：所有主流 Android LLM 项目（PocketPal/ToolNeuron/Meta Llama Stack）均走 JNI，没人用 llama-server HTTP 常驻模式
  2. **HTTP 路径有一连串根因性问题**（见 [深度检查报告 S1/S2/S3](./docs/)）：
     - root 启动的 llama-server 在 `su` 域访问 `/sdcard/` 需 sepolicy 放行 fuse 类型
     - FBE 加密下锁屏首次解锁前 `/sdcard/` 是空 stub，开机自启必然失败
     - Android stock ROM 不带 `curl`，service.sh 健康检查失效
  3. **ToolNeuron 实际是 Kotlin + Compose 全栈**（[原 X2 误判已修正](#-x2-fork-toolneuron-llamaenginekt-重新评估不再废弃)），LlamaEngine.kt 可参考
- **代价**: APK 增 ~50MB（含 arm64-v8a 的 libllama.so + libggml.so），需写 ~1500 行 JNI wrapper（参考 ToolNeuron 已验证的实现）
- **状态**: 已确认
- **修订历史**: 初版 JNI → 第二轮审视改 HTTP（[02_second_review.md 发现 1](./docs/02_second_review.md)，已废弃） → 深度检查后切回 JNI（2026-06-30）
- **关联废弃**: [X7. llama-server HTTP 路径](#-x7-llama-server-http-路径)

### ✅ D8. ASR/TTS 引擎
- **选择**: sherpa-onnx v1.13.3 AAR（官方 Kotlin API）
- **理由**: 已验证 AAR 含 arm64-v8a 的 .so，官方维护
- **日期**: 2026-06-30
- **状态**: 已确认

### ✅ D9. LSPosed 方案
- **选择**: 装现成的 Phantom Mic v2.0，不自写 hook
- **理由**: 已验证是 LSPosed 官方仓库，hook native 层
- **日期**: 2026-06-30
- **状态**: 已确认（带风险）
- **风险**: Phantom Mic v2.0 发布于 2024-07，至 2026-06-30 已近 2 年未更新，HyperOS 2（Android 15）兼容性未验证。风险升级见 [D14](#-d14-phantom-mic-在-hyperos-2-上的兼容性)

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

### 🔴 D14. Phantom Mic 在 HyperOS 2 上的兼容性
- **说明**: 真机未验证；且 Phantom Mic v2.0 发布于 2024-07，**至本次设计冻结（2026-06-30）已近 2 年未更新**，可能已不维护
- **风险等级**: 高
  - 上游可能已停滞，HyperOS 2（Android 15）兼容性更不确定
  - 原设计文档（[00_design_overview.md §10](./docs/00_design_overview.md)、[01_feasibility_verification.md §三 风险 1](./docs/01_feasibility_verification.md)）将此风险标为"中"，已严重低估
- **进入 Phase 1 前的必做检查**:
  1. 去上游仓库 https://github.com/Xposed-Modules-Repo/tn.amin.phantom_mic/releases 看是否有新版本
  2. 看上游 Issues 区是否有 HyperOS 2 / Android 15 兼容性反馈
  3. 若上游确认停滞，需要重新评估是否换备选方案（[06_lspoded_setup.md §6.1](./docs/06_lspoded_setup.md) 列了 XAudioCapture / WhatsMicFix 等候选）
- **状态**: 待 Phase 1 启动前先做上游活跃度复查；真机验证留 Phase 4

### 🟡 D15. 中文唤醒词样本
- **说明**: Phase 3 时需采集 100+ 句"嘿小脑"样本
- **状态**: 待 Phase 3

### ✅ D16. 联网开关语义
- **选择**: 全局开关（设置里一个总开关 + 每个工具子开关）
- **理由**: 简单直接，用户可控；保留"完全离线"默认模式作为卖点
- **状态**: 已确认（Phase 6 设计 [09_phase6_network_tools_design.md](./docs/09_phase6_network_tools_design.md) §4）
- **日期**: 2026-06-30

### ✅ D17. 工具调用机制
- **选择**: 混合方案（关键词正则优先 + 兜底走 LLM function calling）
- **理由**: Qwen2.5-3B 在结构化 function calling 上不稳；纯关键词扩展性差
- **状态**: 已确认（Phase 6 设计 [09_phase6_network_tools_design.md](./docs/09_phase6_network_tools_design.md) §1）
- **日期**: 2026-06-30

### ✅ D18. 位置获取方式
- **选择**: GPS 定位（Android LocationManager）
- **理由**: 最准，root 可强制授定位权限
- **状态**: 已确认（Phase 6 设计 [09_phase6_network_tools_design.md](./docs/09_phase6_network_tools_design.md) §7）
- **日期**: 2026-06-30

### ✅ D19. AppLauncherTool 范围
- **选择**: L1 仅启动 app（不做跳转/填文本/自动点击）
- **理由**: UI 自动化在 MIUI 上极不稳定，L1 最稳，工程量最小
- **状态**: 已确认（Phase 7 设计 [10_phase7_phone_control_design.md](./docs/10_phase7_phone_control_design.md) §3）
- **日期**: 2026-06-30

### ✅ D20. 多模态视觉 API
- **选择**: 默认 GLM-4V（智谱），备选 GPT-4V/Claude/Qwen-VL
- **理由**: 国内访问稳，中文好，多模态质量不错
- **状态**: 已确认（Phase 7 ScreenCaptureTool + Phase 9 多模态共用）
- **日期**: 2026-06-30

### ✅ D21. 模型存储路径（2026-06-30 新增）
- **选择**: **Device Encrypted (DE) 存储区 + Direct Boot 模式**
  - 路径：`/data/user_de/0/com.mibrain/files/models/`
  - Direct Boot 下可读（不依赖用户首次解锁）
  - appops + KSU 配合可让 root 服务在锁屏前访问
- **选择理由**:
  1. 解决 S2 FBE 加密问题：CE 加密区在用户首次解锁前不可读，DE 加密区在开机即可读
  2. 解决 S1 跨 SELinux 域问题：DE 区属于 `app_data_file` 类型，JNI 调用（D7）不跨域
  3. Direct Boot 让 Phase 4 "锁屏唤醒" 验收标准在加密设备上也能达成
- **首次启动流程**: APK 首次启动 → 检测 DE 区模型 → 不存在则触发下载器（带进度 UI + 断点续传 + SHA256 校验）
- **关联**: 解决深度检查 S1 + S2
- **状态**: 已确认
- **日期**: 2026-06-30

### ✅ D22. ASR/TTS 模型许可友好化（2026-06-30 新增）
- **选择**: 换用 Apache 2.0 许可的 sherpa-onnx 官方模型，弃用 CC BY-NC / CC BY-NC-ND 模型
- **修订原因**: 深度检查 S7 发现原 paraformer (CC BY-NC 4.0) + aishell3 (CC BY-NC-ND 4.0) 与项目 Apache 2.0 许可冲突，分发时违法
- **新模型选型**（待 Phase 2 实施时确认具体版本，以下为候选）：
  | 能力 | 候选模型 | 许可 |
  |---|---|---|
  | 流式 ASR | sherpa-onnx-streaming-zipformer-bilingual-zh-en | Apache 2.0 |
  | 中文 TTS | sherpa-onnx-vits-zh-ll（替代 aishell3）| Apache 2.0 |
  | 英文 TTS | vits-vctk（ sherpa-onnx 官方）| Apache 2.0 |
  | VAD | silero_vad（sherpa-onnx 自带）| Apache 2.0 |
  | KWS | sherpa-onnx-keyword-spotting-zh-vgg | Apache 2.0 |
- **代价**: 中文 TTS 自然度可能略降（aishell3 训练数据更全），但避免法律风险
- **关联**: 解决深度检查 S7；与 [D23](#-d23-唤醒词改用-sherpa-onnx-kws) 一起统一 sherpa-onnx 全栈
- **状态**: 已确认
- **日期**: 2026-06-30

### ✅ D23. 唤醒词改用 sherpa-onnx KWS（2026-06-30 新增）
- **选择**: **弃用 openWakeWord，改用 sherpa-onnx 自带的 KWS 模块**（keyword spotting）
- **修订原因**:
  1. sherpa-onnx v1.13.3 已内置 KWS 能力（含 zh-vgg 模型），无需另装 openWakeWord + 自写 Kotlin wrapper
  2. 统一技术栈：sherpa-onnx 一个 AAR 同时提供 ASR + TTS + VAD + KWS 四件套，减少依赖
  3. openWakeWord 自写 wrapper 是原 [00_design_overview.md §6](./docs/00_design_overview.md) 列的"重写为 Kotlin"，与 [D8](#-d8-asrtts-引擎) sherpa-onnx 选型重复
- **代价**: 备选唤醒词模型可能没有 `hey_jarvis`，需用 sherpa-onnx 官方示例词或自训
- **关联**: [D2](#-d2-唤醒词策略) 的"openWakeWord 没有现成中文模型"理由随此决策失效；与 [D22](#-d22-asrtts-模型许可友好化) 统一 sherpa-onnx 全栈
- **状态**: 已确认
- **日期**: 2026-06-30

## 已废弃决策

### ❌ X1. Go daemon
- **说明**: 之前考虑过 Go daemon + NDK 交叉编译
- **废弃原因**: Android 上 `CGO_ENABLED=0` DNS 会崩，NDK 编译门槛高
- **废弃日期**: 2026-06-30

### ✅ X2. fork ToolNeuron LlamaEngine.kt（2026-06-30 重新评估，不再废弃）
- **原废弃原因（误判）**: 之前判断 "ToolNeuron 用的是自己写的 C++ + JNI，不是纯 Kotlin"
- **重新评估结论**: 开源调研（2026-06-30）证实 ToolNeuron 是 **Kotlin + Jetpack Compose + llama.cpp + sherpa-onnx + Stable Diffusion + RAG 全栈同构**（258★，2026-05 仍活跃），LlamaEngine.kt 可作为参考样板
- **新结论**: 不直接 fork 整个 ToolNeuron，但**强烈推荐作为代码参考样板**，特别是 LlamaEngine.kt 的 JNI 调用范式、模型加载/卸载、AES-256-GCM 加密存储等工程实践
- **关联决策**: 见 [D7](#-d7-推理后端2026-06-30-修订切回-jni)

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

### ❌ X6. 插件系统（用户写 Tool.kt 热加载）（2026-06-30 归档修正）
- **说明**: Phase 8 候选能力 Cap 2，用户决策跳过
- **废弃原因**: DexClassLoader + 反射的安全风险大，沙箱实现复杂，不符合当前阶段优先级
- **废弃日期**: 2026-06-30
- **替代方案**: 用户/第三方通过 Phase 8 Cap 1 的本地 API 暴露（8080 端口）或 Cap 4 MCP 协议扩展

### ❌ X7. llama-server HTTP 路径（2026-06-30 废弃）
- **说明**: 第二轮审视（[02_second_review.md 发现 1](./docs/02_second_review.md)）曾将推理后端从 JNI 改为 llama-server HTTP 二进制 + KSU 自启，原 D7 采纳
- **废弃原因**: 深度检查 + 开源调研后发现该路径在 Android 上有根因性问题：
  1. **无任何先例**：所有主流 Android LLM 项目（PocketPal/ToolNeuron/Meta Llama Stack）均走 JNI
  2. **S1**: root 启动的 llama-server 在 `su` SELinux 域访问 `/sdcard/` 模型需 sepolicy 放行 fuse 类型，未定义
  3. **S2**: FBE 加密下锁屏首次解锁前 `/sdcard/` 是空 stub，service.sh 开机自启必然失败
  4. **S3**: Android stock ROM 不带 `curl`，service.sh 健康检查失效，watchdog 误判反复重启
- **替代方案**: 见 [D7](#-d7-推理后端2026-06-30-修订切回-jni)，切回 llama.android JNI 模块
