# MiBrain 可行性复审 + 分阶段实施计划（三轮深度复审版）

> 本文档在 Phase 0 设计冻结后，对全部关键依赖做**三轮深度可行性复审**，并基于复审结果给出严谨的分阶段实施计划。
>
> **复审历史**：
> - **第一轮（2026-06-30）**：3 个并行 subagent 实测 GitHub / HuggingFace / 官方文档站，发现 E1-E6 严重错误 + W1-W3 估算偏差 + R1-R6 风险。
> - **第二轮（2026-06-30）**：基于第一轮发现修订 D7/D22/D23 + 新增 D24-D27 + E7-E13 + F1-F2 二次审视。
> - **第三轮（2026-06-30）**：主会话直接用 WebSearch 独立核实 8 项关键事实，纠正前轮总结中 LSPosed Vector 发布日期错误（2026-03-31 → 实际 2026-04-21），补 NON-COMMERCIAL 标注，新增 D28-D31 + R10-R13。
>
> 每个验证项明确标 ✅ 已验证 / ⚠️ 部分验证有疑问 / ❌ 与文档不符。
>
> **三轮复审日期**：2026-06-30

---

## 一、复审汇总

### 1.1 已确认可行 ✅（无需调整）

| 项 | 验证结论 | 来源 |
|---|---|---|
| llama.cpp b9830 | 真实 release tag，MIT 许可 | [release b9830](https://github.com/ggml-org/llama.cpp/releases/tag/b9830) |
| llama.android 官方 JNI 模块 | 真实存在，含完整 Gradle 工程 + Kotlin Flow 流式 token | [examples/llama.android](https://github.com/ggml-org/llama.cpp/tree/master/examples/llama.android) |
| android-arm64 tarball | 真实存在，含 libllama.so + libggml.so + bin + include | release 直链 |
| NDK 编译文档 | 齐全，含 CMake 完整参数 | docs/android.md |
| sherpa-onnx v1.13.3 | 真实存在（2026-06-15 发布），Apache 2.0 | [release v1.13.3](https://github.com/k2-fsa/sherpa-onnx/releases/tag/v1.13.3) |
| AAR 含 arm64-v8a .so | libsherpa-onnx-jni.so (3.7M) + libonnxruntime.so (15M) | 官方构建产物 |
| sherpa-onnx Kotlin API 四件套 | ASR + TTS + VAD + KWS 全覆盖 | 官方矩阵 |
| 8+ Gen 1 上 ASR RTF | 实测 < 0.05，可流式实时 | voiceping.net 基准 |
| Qwen2.5-1.5B GGUF HF 链接 | 真实可达，q4_k_m 文件 ~1.5GB | 多源交叉验证 |
| Qwen2.5-1.5B 许可 | **Apache License 2.0**（非 Research License），允许商用 | php.cn 商用合规指南 |
| Qwen2.5-1.5B 中文能力 | 29+ 语言，C-Eval 显著优于 1.5-1.8B | 官方文档 |
| silero_vad | sherpa-onnx 自带，MIT 许可，629KB | 官方 VAD 文档 |
| KWS 自定义词无需训练 | open-vocabulary，text2token 即可，"嘿小脑"直接可用 | sherpa-onnx KWS 文档 |
| Direct Boot 官方支持 | directBootAware=true 可让 FGS 解锁前启动 | developer.android.com |
| AudioRecord Direct Boot 可工作 | BCR（Basic Call Recorder）已验证 | BCR 项目 |
| ToolNeuron 仓库活跃 | 429★，2026-05-18 仍活跃，Kotlin + Compose | GitHub API |

### 1.2 第一轮发现的严重错误 ❌（已全部修复）

| # | 错误 | 真实情况 | 影响范围 |
|---|---|---|---|
| E1 | KWS 模型名 `sherpa-onnx-keyword-spotting-zh-vgg` | **不存在**！真实为 `sherpa-onnx-kws-zipformer-wenetspeech-3.3M-2024-01-01`（中文）或 `sherpa-onnx-kws-zipformer-zh-en-3M-2025-12-20`（中英双语，更新更小） | 全项目所有提到 KWS 模型的文档 |
| E2 | ToolNeuron `LlamaEngine.kt` | **不存在**！真实文件为 `InferenceService.kt` (57.8KB) + `InferenceClient.kt` (43.9KB)，位于 `app/src/main/java/com/dark/tool_neuron/service/inference/` | 全项目所有引用 LlamaEngine.kt 的文档 |
| E3 | llama.android 路径 `tree/master/llama.android` | **404**！正确路径 `tree/master/examples/llama.android/` | README / NOTICES / 00 / 01 / 02 / DECISIONS |
| E4 | vits-zh-ll 声称 Apache 2.0 | HF 卡 metadata 缺失，**官方未声明许可**，训练数据来源不明 | D22 / 00 / 01 / 05 / 07 / NOTICES |
| E5 | Android opencl-adreno 包 | **仅 Windows 版**（`-bin-win-opencl-adreno-arm64.zip`）；Android Adreno 加速需自行用 Snapdragon 工具链编译 | 01_feasibility_verification.md §2.1 |
| E6 | streaming-zipformer-bilingual-zh-en 命名 | 真实文件名带日期后缀 `-2023-02-20` | 多份文档 |

### 1.3 估算偏差 ⚠️（已全部修复）

| # | 文档原值 | 真实值 | 影响评估 |
|---|---|---|---|
| W1 | libllama.so + libggml.so ~50MB | **~30MB**（日本 Pasona 实测） | D7 代价估算偏高，APK 体积比预期小 |
| W2 | ToolNeuron 258★ | **429★**（GitHub API 实时） | 借鉴价值被低估 |
| W3 | ToolNeuron Apache 2.0 | **MIT**（2026-04-22 从 GPLv3 切到 MIT） | NOTICES 已正确，但其他文档可能有误 |

### 1.4 第一轮风险升级与新发现（已修复）

| # | 风险 | 评级 | 缓解 |
|---|---|---|---|
| R1 | **TTS 无 chunk 回调**：sherpa-onnx TTS 是 OfflineTts（一次性生成完整 PCM），无官方流式回调 | 中 | Phase 2 / Phase 9 G6 字幕同步需自切 PCM 数组成 chunk 喂 AudioTrack |
| R2 | **Qwen2.5-1.5B function calling brittle**：BFCL v3 实测直答准确率 44%，需 32-token CoT 才到 64%，长 CoT 反崩到 25% | 高 | Phase 6 工具调用需 32-token CoT prompt 或换 3B；M1.5B 复杂工具调用不可靠 |
| R3 | **Direct Boot + sherpa-onnx 组合未公开验证**：理论可行但无先例，4 个子项需 PoC | 高 | Phase 1 启动前必做 PoC |
| R4 | **Phantom Mic 高风险评级准确甚至偏保守**：最后 release 2024-07-24，近 2 年未更新，无 HyperOS 3 / Android 15 兼容性反馈，**无对等替代品**；**第四轮 web 核实确认上游已停滞 23 个月**（自 2024-07-24 后零更新） | 高 | D14 已准确，无新缓解方案；Phase 4 真机验证为硬关卡；**第四轮补降级方案**（appops + 双触发兜底 / WhatsMicFix 改造 / 放弃锁屏唤醒） |
| R5 | **Qwen 商标限制**：Apache 2.0 允许商用但 "Qwen" 商标不可用于衍生品市场宣传 | 低 | README / UI 不写 "Powered by Qwen"，改写 "使用 Qwen2.5 模型" |
| R6 | **sherpa-onnx v1.13.3 PyPI wheel 未上传**：仅 GitHub 源码 + 二进制资产，PyPI 仍是 1.13.2 | 低 | Android APK 直接用 AAR，不依赖 PyPI |

### 1.5 第二轮 + 第三轮 web 核实补正（已修复）

> 第二轮 + 第三轮通过 WebSearch 独立核实 8 项关键事实，纠正前轮总结中的事实错误，并补 NON-COMMERCIAL 标注。

#### 1.5.1 第二轮发现（E7-E13 + F1-F2 二次审视）

| # | 错误 / 风险 | 真实情况 | 修订决策 |
|---|---|---|---|
| E7 | NanoHTTPD 推荐 | **last commit ~2 年前**，仅 HTTP 1.1，无内置 SSE，162 open issues 无人响应，实质已停止维护 | [D28](../DECISIONS.md)：改用 Ktor |
| E8 | MCP Kotlin SDK "Android 支持未明确" | 实际由 modelcontextprotocol 官方组织维护，1393★，0.13.0，已有第三方 `kaeawc/android-mcp-sdk` 作为 Android PoC | 11 §3.1 修订 |
| E9 | 微信 A2A "调研是否对第三方 APK 开放" | 已核实**不对第三方 APK 开放**，仅对手机系统级 AI 助手（华为/小米/荣耀/OPPO/vivo）开放 | [D29](../DECISIONS.md)：移除该路径 |
| E10 | 1.5B 内存峰值 6.77GB | 算术错误。修正后：1.5B 串行峰值 **~7.5GB**，最坏叠加峰值 **7.89GB**，headroom 仅 **0.11-0.5GB**；3B 峰值 **8.59GB**，**8GB 设备必 OOM** | [D30](../DECISIONS.md)：3B 8GB 设备不可用 |
| E11 | matcha-icefall-zh-baker Apache 2.0 | 模型权重 Apache 2.0，但训练数据 Data-Baker 是 **NON-COMMERCIAL ONLY**，与项目 Apache 2.0 分发不兼容，**只能本地测试，不能分发** | [D22](../DECISIONS.md) 修订 |
| E12 | kws-zipformer-wenetspeech Apache 2.0 | 模型权重 Apache 2.0，但训练数据 WenetSpeech 论文原文明确写 "**licensed for non-commercial usage under CC-BY 4.0**" | [D23](../DECISIONS.md) 修订 |
| E13 | 8 步白名单 | HyperOS 2.0+ 新增"应用智能休眠"开关，电池白名单不豁免，必须单独关闭 | [D31](../DECISIONS.md)：9 步白名单 |
| F1 | LSPosed Vector 前轮总结写 "2026-03-31 发布" | 真实发布日期：v2.0.1 2026-04-21 / v2.0.2 2026-05-05 / **v2.0.3-7716 2026-05-20**，支持 Android 8.1-17 Beta 3 | [D9](../DECISIONS.md)：LSPosed Vector 复活记录 |
| F2 | HyperOS 2 | HyperOS 3 国行版已对 K50U 推送：OS3.0.1.0.VLFCNXM 2026-01-20 OTA / OS3.0.2.0.VLFCNXM 2026-04-13 Fastboot | [D6](../DECISIONS.md)：HyperOS 3 升级 |

#### 1.5.2 第三轮新增风险（R10-R13）

| # | 风险 | 评级 | 缓解 |
|---|---|---|---|
| R10 | **验证码短信被 TTS 朗读明文**：用户在公共场合，6 位验证码短信到达，TTS 把明文播报出来，他人可窃取 | 高 | 默认识别验证码模式 → 仅 TTS 播报"你有一条验证码短信"，不朗读明文；锁屏状态不朗读敏感通知 |
| R11 | **银行/支付通知被朗读**：余额变动、转账短信、信用卡账单 | 中 | 默认对"银行/转账/余额"等关键词做敏感过滤；默认仅朗读 appName，不朗读 text |
| R12 | **GLM-4V API 需实名认证**：智谱开放平台强制手机号 + 中国大陆身份证实名，境外用户无法直接实名 | 中 | UI 首次启用必须弹出实名认证说明；备选 GPT-4V / Claude 3.5（境外可付费但需代理） |
| R13 | **base64 流量未量化**：原图 + 多轮对话累计可达数十 MB，4G 网络延迟可能 5-15s，触发 HTTP 30s 超时 | 中 | MVP 默认"高压缩"档位（720p JPEG q70，单张 ~270-540KB）；移动数据自动降级到"中压缩"（可选开关） |

### 1.5.3 第四轮 web 核实补正（2026-06-30，本轮新增）

> 第四轮通过 WebSearch / WebFetch 独立核实 8 项关键依赖最新状态，发现 4 项需修订（F3-F7）。

| # | 错误 / 风险 | 真实情况 | 修订决策 |
|---|---|---|---|
| F3 | **llama.cpp 锁定版本 b9844 不存在** | 2026-06-30 时点 GitHub 最新 release 实为 **b9830**（2026-06-28 发布），b9844 尚未发布。若 CI 拉 `releases/tag/b9844` 会 404 失败 | [D7](../DECISIONS.md)：版本号 b9844 → b9830；全项目 replace_all |
| F4 | **LSPosed Vector 来源归属未明确** | 当前活跃维护的 fork 位于 **`JingMatrix/Vector`** 仓库（非原 `LSPosed/LSPosed`） | [D9](../DECISIONS.md)：补来源归属，Phase 1 下载以此仓库为准 |
| F5 | **Phantom Mic 上游已确认停滞** | 自 2024-07-24 v2.0 后**再无任何新版本发布**，至 2026-06-30 已停滞 **23 个月**，上游确认无后续维护 | [D14](../DECISIONS.md)：补降级方案（appops + 双触发兜底 / WhatsMicFix 改造 / 放弃锁屏唤醒） |
| F6 | **WenetSpeech 许可表述自相矛盾**（修正原 E12 表述） | OpenSLR 官方数据集页标注 "License: CC BY 4.0"（允许商用），但论文写 "non-commercial usage under CC-BY 4.0"（自相矛盾）。原表述"CC-BY 4.0 non-commercial"不准确 | [D23](../DECISIONS.md)：修正表述为"许可表述自相矛盾致商用存疑"，风险等级保持高 |
| F7 | **matcha-icefall-zh-baker 不是合规替代**（确认 E11 判断） | matcha-icefall-zh-baker 基于 Data-Baker 语料训练，Data-Baker 明确 NON-COMMERCIAL，**不是 vits-zh-ll 的合规替代方案**。需另寻基于 LibriTTS / Common Voice（CC0 / CC BY 4.0）的中文模型 | [D22](../DECISIONS.md)：补"matcha 不是合规替代"明确结论 |

**第四轮无修订项**（核实通过）：
- sherpa-onnx v1.13.3：git tag 2026-06-15 存在（仅有 git tag 无正式 Release notes，不影响 AAR 下载）
- HyperOS 3 K50U 推送：OS3.0.1.0.VLFCNXM 2026-01-20 OTA / OS3.0.2.0.VLFCNXM 2026-04-13 Fastboot，与假设完全一致
- Qwen2.5-1.5B function calling：44%/64% 数值无法从公开来源直接核实，但与"小模型低于 50%"行业认知一致，建议 Phase 1 自测验证

---

## 二、分阶段实施计划

> 基于三轮复审结果，将原 Phase 1-10 重组为 12 个 Stage，每个 Stage 设明确**准入条件** / **交付物** / **准出条件** / **风险关卡**。
>
> 关键原则：
> 1. **未通过可行性补救（Stage 0）不启动 Phase 1**
> 2. **未通过 PoC（Stage 1）不写正式代码**
> 3. **未通过真机验证（Phase 4）不发布**
> 4. **每 Stage 设回退路径**，避免卡死
> 5. **Phase 5 发布前必须解决 D22/D23 训练数据 NON-COMMERCIAL 遗留**（TTS + KWS 模型许可）

### Stage 0：可行性补救（三轮修复整合，预计 1-2 天）

**目标**：修复第一轮 6 项严重错误 + 第二轮 + 第三轮 13 项发现 + NON-COMMERCIAL 标注，让所有文档回到可信状态。

**交付物（按三轮发现顺序整合）**：

#### 第一轮修复（已在前两轮会话完成）
- [ ] 全项目搜索 `keyword-spotting-zh-vgg` → 替换为 `kws-zipformer-wenetspeech-3.3M-2024-01-01`（中文默认）或 `kws-zipformer-zh-en-3M-2025-12-20`（中英双语）
- [ ] 全项目搜索 `LlamaEngine.kt`（在 ToolNeuron 引用语境下）→ 改为 `InferenceService.kt + InferenceClient.kt`
- [ ] 全项目搜索 `tree/master/llama.android` → 改为 `tree/master/examples/llama.android`
- [ ] 修订 D22：vits-zh-ll 改为"社区贡献，许可未明确声明"，**不可声称 Apache 2.0**
- [ ] 修订 01_feasibility_verification.md §2.1：删除 "Android opencl-adreno-arm64" 表述
- [ ] 修订 streaming-zipformer 文件名：补 `-2023-02-20` 后缀
- [ ] 修订 libllama.so 体积：~50MB → ~30MB
- [ ] 修订 ToolNeuron star 数：258 → 429
- [ ] 新增风险表条目：TTS chunk / Qwen 1.5B function calling / Direct Boot 未验证
- [ ] 在 README.md 加 "Qwen 商标不可用于衍生品宣传" 提示

#### 第二轮 + 第三轮修复（前轮已完成）
- [x] **[E10]** 修订 [03_architecture_detail.md §6](./03_architecture_detail.md) 内存算法：1.5B 串行峰值 ~7.5GB / 3B 峰值 8.59GB（必 OOM）
- [x] **[E7]** 修订 [11_phase8_platform_design.md §1.2/§1.3/§1.5](./11_phase8_platform_design.md) HTTP server：NanoHTTPD → Ktor
- [x] **[E9]** 修订 [10_phase7_phone_control_design.md §2.3.2](./10_phase7_phone_control_design.md) 微信 A2A 移除
- [x] **[E13]** 修订 [05_deploy_guide.md §5](./05_deploy_guide.md) 白名单 8 步 → 9 步（新增应用智能休眠）
- [x] **[F1]** 修订 [DECISIONS.md D9](../DECISIONS.md) LSPosed Vector 复活记录（v2.0.3-7716, 2026-05-20）+ D14 HyperOS 3 兼容性
- [x] **[F2]** 修订 [DECISIONS.md D6](../DECISIONS.md) HyperOS 3 升级 + [05_deploy_guide.md §0](./05_deploy_guide.md) 前置条件
- [x] **[E11]** 修订 [DECISIONS.md D22](../DECISIONS.md) matcha-icefall-zh-baker 标为 NON-COMMERCIAL（Data-Baker 训练数据）
- [x] **[E12]** 修订 [DECISIONS.md D23](../DECISIONS.md) kws-zipformer-wenetspeech 标为 NON-COMMERCIAL（WenetSpeech 训练数据）
- [x] **[E8]** 修订 [11_phase8_platform_design.md §3.1](./11_phase8_platform_design.md) MCP Kotlin SDK 归属修正 + kaeawc/android-mcp-sdk PoC
- [x] **[R10/R11]** 修订 [10_phase7_phone_control_design.md §2.4.1](./10_phase7_phone_control_design.md) 敏感通知朗读风险
- [x] **[R12]** 修订 [12_phase9_multimodal_design.md §4.1](./12_phase9_multimodal_design.md) GLM-4V 实名认证
- [x] **[R13]** 修订 [12_phase9_multimodal_design.md §3.3.1](./12_phase9_multimodal_design.md) base64 流量量化
- [x] **[D28-D31]** 新增决策到 [DECISIONS.md](../DECISIONS.md)
- [x] **[06_lspoded_setup.md]** §1 + §6.1 更新 LSPosed Vector + HyperOS 3

#### 第四轮修复（本轮新增，2026-06-30）
- [x] **[F3]** 修订 [DECISIONS.md D7](../DECISIONS.md) llama.cpp 版本号 b9844 → b9830（b9844 未发布）；全项目 7 个文件 replace_all
- [x] **[F4]** 修订 [DECISIONS.md D9](../DECISIONS.md) LSPosed Vector 补来源归属（`JingMatrix/Vector` fork）
- [x] **[F5]** 修订 [DECISIONS.md D14](../DECISIONS.md) Phantom Mic 确认停滞 23 个月 + 补降级方案（4 条路径）
- [x] **[F6]** 修订 [DECISIONS.md D23](../DECISIONS.md) WenetSpeech 许可表述修正为"自相矛盾致商用存疑"
- [x] **[F7]** 修订 [DECISIONS.md D22](../DECISIONS.md) 明确 matcha-icefall-zh-baker 不是合规替代
- [x] 全项目 HyperOS 2 → HyperOS 3（13 个文件）
- [x] 全项目旧内存数字（6.77GB/1.23GB/7.67GB/330MB）→ 第三轮重算后数字（7.5GB/0.5GB/7.89GB/0.11GB）
- [x] 全项目 8 步 → 9 步白名单（6 个文件）
- [x] 全项目旧决策计数（23 项/17 条）→ 31 项
- [x] [README.md](../README.md) 修订说明 blockquote 补第四轮 F3-F7 + Root 框架表 LSPosed → LSPosed Vector + F4 来源
- [x] [00_design_overview.md](./00_design_overview.md) §6/§10 修订 b9830 + HyperOS 3 + 内存数字 + §11 决策清单 3B 可选 → 不可用
- [x] [01_feasibility_verification.md](./01_feasibility_verification.md) §三风险 1/2 + §七结论修订 + §八决策清单 3B 可选 → 不可用
- [x] [03_architecture_detail.md](./03_architecture_detail.md) b9830 replace_all + §6 "3B 质量优先模式" → 已废弃
- [x] [05_deploy_guide.md](./05_deploy_guide.md) §故障排查 3B 切换 → 不可切
- [x] [06_lspoded_setup.md](./06_lspoded_setup.md) §1 LSPosed Vector Releases 链接 `LSPosed/LSPosed` → `JingMatrix/Vector` + §6.1 风险升级表述 + §6.1 降级方案补方案 D 放弃锁屏唤醒
- [x] [07_troubleshooting.md](./07_troubleshooting.md) §7.1/7.2 3B 质量优先 → 不可用
- [x] [09_phase6_network_tools_design.md](./09_phase6_network_tools_design.md) §2 3B 可选 → 不可用 + §11 Q4 3B 真机验证 → 不可引导用户切 3B
- [x] [13_phase10_ux_enhancements_design.md](./13_phase10_ux_enhancements_design.md) §2.5/§11/§13 G2 儿童模式 3B 备选 → 不可用
- [x] [app/README.md](../app/README.md) / [ksu_module/README.md](../ksu_module/README.md) / [scripts/README.md](../scripts/README.md) 旧 HTTP/llama-server/3B 路径修订（详见下文 §三 CI 方案）
- [x] [.github/ISSUE_TEMPLATE/bug_report.md](../.github/ISSUE_TEMPLATE/bug_report.md) / [CONTRIBUTING.md](../CONTRIBUTING.md) / [docs/README.md](./README.md) / [02_second_review.md](./02_second_review.md) HyperOS 2 → 3 + 其他旧引用
- [x] [third_party/NOTICES.md](../third_party/NOTICES.md) b9844 → b9830 + Phantom Mic 风险表述补"第四轮确认停滞 23 个月"
- [x] [DECISIONS.md D1](../DECISIONS.md) 备选条件 3B "质量优先可选项" → "8GB 设备不可用，仅 12GB+ 或 Phase 11+ GPU 加速后可选"
- [x] [DECISIONS.md D25](../DECISIONS.md) 代价项 3B "7.67GB 峰值" → "8.59GB 必 OOM 8GB 设备不可用"

**准出条件**：所有文档无幻觉引用，所有依赖链接可达，所有许可声明准确（含训练数据 NON-COMMERCIAL 标注）。

**回退路径**：无（这是补救，不通过则整个项目可信度崩塌）。

### Stage 1：关键 PoC（Phase 1 启动前必做，预计 3-5 天）

**目标**：用最小代码验证三个高风险技术点，避免 Phase 1 写了一半发现根因性问题。

**3 个 PoC 并行**：

#### PoC-A：Direct Boot + sherpa-onnx 加载链路（硬关卡）
- 写一个最小 APK：Application directBootAware + ForegroundService directBootAware
- 在锁屏前重启手机，验证：
  1. `ACTION_LOCKED_BOOT_COMPLETED` 广播能收到
  2. `Context.createDeviceProtectedStorageContext().openFileInput()` 能读 DE 区文件
  3. `System.loadLibrary("sherpa-onnx-jni")` 在 Direct Boot 下能加载 .so
  4. sherpa-onnx OfflineRecognizer 能从 DE 路径加载模型并完成一次推理
  5. **HyperOS 3** 的"超级省电"不在解锁前杀 FGS
- **不通过** → 退路：放弃 Direct Boot 模式，回退到 "用户首次解锁后才启动"（牺牲锁屏唤醒能力，但保留其他所有功能）

#### PoC-B：WhatsMicFix 改造验证（硬关卡，第六轮 R5 重定义；原 Phantom Mic 兼容性验证已合并到此）

> **第六轮 R5 订正**：原 PoC-B 定义为 "Phantom Mic 在 HyperOS 3 上的兼容性验证"，但 [D14](../DECISIONS.md) 第六轮 R5 已确认 Phantom Mic 上游自 2024-07-24 v2.0 后停滞 23 个月且无新版，且 WhatsMicFix-LSPosed 第六轮 web 核实仍活跃维护（commit `0715c57` 2026-03-15 / 75 commits / 4 tags 含 v1.4 2025-10-28），双 scope 注入架构可改造。故 PoC-B 重定义为 WhatsMicFix 改造验证，Phantom Mic 兼容性验证合并到此 PoC 的"对照测试"分支（仍可单独跑作为基线，但不再是首选路径）。

- 装 LSPosed Vector v2.0（[D9](../DECISIONS.md)，第六轮 R5 订正版本号；原 v2.0.3-7716 无法对应公开 release）
- **主路径（WhatsMicFix 改造）**：
  1. clone https://github.com/D4vRAM369/WhatsMicFix-LSPosed
  2. 改 scope 配置（`com.whatsapp` → `com.mibrain`）+ 改目标包名常量
  3. 编译 APK（Android Studio + Gradle）
  4. 装到 K50U + 在 LSPosed Vector 启用作用域 + 重启
  5. 锁屏后 `AudioRecord.read()` 是否返回非零字节 ≥ 80%（采样 30s × 3 次取最小值，`adb logcat | grep AudioRecord`）
- **对照路径（Phantom Mic 兼容性基线，可选）**：装 Phantom Mic v2.0 + LSPosed Vector 启用 + 作用域勾选测试 APK + 锁屏后 `AudioRecord.read()` 是否返回真实数据
- **同时验证**：LSPosed Vector 框架本身在 HyperOS 3 K50U 上是否激活（无框架 = 模块也无用，[D14](../DECISIONS.md)）
- **不通过** → 退路（按 [D14](../DECISIONS.md) 第六轮 R5 排序）：
  1. 评估 XAudioCapture 作为更轻量备选（hook 的是播放侧，对麦克风场景适配性未验证）
  2. 退到 appops + 双触发兜底（[06_lspoded_setup.md §6.1](./06_lspoded_setup.md) 方案 C；**注意**：appops 仅改 OS 权限位，不绕 MIUI AudioRecord.cpp 内部检查，大概率仍返回 0 字节）
  3. 最坏情况：放弃锁屏唤醒，仅亮屏可用

#### PoC-C：JNI wrapper 编译可行性
- 拉 llama.cpp b9830 源码
- 按 docs/android.md 的 NDK CMake 参数编译 libllama.so + libggml.so
- 集成到最小 Android Studio 工程
- 调 llama.android examples 里的 InferenceEngine 加载 1.5B Q4_K_M
- 跑一次 `complete("hello")` 看是否成功
- **不通过** → 退路：用官方 android-arm64 tarball 解包取 .so 而非自编译

**准出条件**：3 个 PoC 全部通过，或部分通过且有明确退路。

**回退路径**：见各 PoC 退路。

### Stage 2：Phase 1 MVP 单链路（预计 2-3 周）

**准入**：Stage 0 + Stage 1 全部通过。

**目标**：APK 能跑：UI 一个按钮 → JNI 调 LlamaEngine → 显示文本回复。

**交付物**：
- [ ] Kotlin + Compose APK 骨架（参考 [ToolNeuron](https://github.com/Siddhesh2377/ToolNeuron) `InferenceService.kt` + [llama.android](https://github.com/ggml-org/llama.cpp/tree/master/examples/llama.android) `InferenceEngine`）
- [ ] JNI wrapper：`app/src/main/jni/`（CMakeLists.txt + llama_engine.cpp + .h）
- [ ] **LlamaEngine.kt**（项目自命名，与 ToolNeuron 实际文件名 InferenceService.kt 区分；明确文档说明"参考 InferenceService.kt 实现思路"而非"fork LlamaEngine.kt"）
- [ ] AndroidManifest.xml：`directBootAware=true` + `foregroundServiceType="microphone"` + `largeHeap=true`
- [ ] ModelManager：DE 区模型下载 + SHA256 校验 + 断点续传
- [ ] 一个静态对话界面（按钮触发，非语音）
- [ ] 编译指南 [04_build_guide.md](./04_build_guide.md)

**准出条件**：
- APK 能在 K50U 上跑通静态对话
- 1.5B Q4_K_M 加载耗时 < 10s
- 单次推理速度 > 5 tok/s
- DE 区模型 SHA256 校验通过
- 锁屏前 ForegroundService 不被杀（10 分钟测试）

**回退路径**：3B 模型 8GB 设备不可用（[D30](../DECISIONS.md)），仅 1.5B 可选；JNI 编译失败用 tarball .so。

### Stage 3：Phase 2 语音链路（预计 2-3 周）

**准入**：Stage 2 准出。

**目标**：完整对话循环 IDLE→LISTENING→THINKING→SPEAKING→COOLDOWN→IDLE。

**交付物**：
- [ ] sherpa-onnx StreamingAsr 集成（zipformer-bilingual-zh-en-2023-02-20）
- [ ] sherpa-onnx OfflineTts 集成（vits-zh-ll；第六轮 R1 订正：有 `generateWithCallback` 进度回调可用于流式播放，非"无 chunk 回调"；许可遗留待 Phase 5 解决）
- [ ] sherpa-onnx `Vad` 集成（silero_vad；第六轮 R1 订正类名，原 `VoiceActivityDetector` 不存在）
- [ ] 前台服务 + WakeLock + 状态机 CAS
- [ ] 录音权限请求 UI

**关键风险**：
- TTS 无 chunk 回调 → MVP 用"先合成完整 PCM 再播"模式（延迟略大但简单）
- 若需"边合成边播"，Kotlin 端按 200ms 切 PCM 数组成 chunk
- **TTS 模型许可遗留**：vits-zh-ll 训练数据来源不明，matcha-icefall-zh-baker 训练数据 Data-Baker NON-COMMERCIAL（[D22](../DECISIONS.md)），Phase 5 发布前必须解决

**准出条件**：
- 完整对话循环跑通
- ASR RTF < 0.5（远低于 1.0）
- TTS 首音延迟 < 2s
- VAD 误判率 < 10%

**回退路径**：TTS 太慢降级为"显示文字回复 + 静音"。

### Stage 4：Phase 3 唤醒词（预计 1-2 周）

**准入**：Stage 3 准出。

**目标**：sherpa-onnx KWS 持续监听 + 唤醒触发对话。

**交付物**：
- [ ] sherpa-onnx KeywordSpotter 集成
- [ ] KWS 模型选 `kws-zipformer-wenetspeech-3.3M-2024-01-01`（中文默认，**训练数据 WenetSpeech NON-COMMERCIAL**，[D23](../DECISIONS.md)；Phase 5 发布前确认许可）
- [ ] text2token 生成 "嘿小脑" / "你好小脑" keywords.txt
- [ ] 调参：boosting_score 1.5-3.5 + threshold 0.25-0.6
- [ ] KWS 与 ASR 切换不冲突（IDLE 用 KWS，LISTENING 用 ASR）
- [ ] 中文唤醒词样本采集（[D15](../DECISIONS.md)）

**准出条件**：
- 唤醒率 > 80%（10 次说 8 次以上触发）
- 误唤醒 < 1 次/小时
- KWS 占用 CPU < 5%

**回退路径**：自训中文效果差 → MVP 用英文 `hey_jarvis`（[D2](../DECISIONS.md)）+ 桌面快捷方式兜底；若 WenetSpeech 许可问题最终未解决，**改用 gigaspeech KWS**（避开 WenetSpeech 数据）。

### Stage 5：Phase 4 稳定性（预计 2-3 周，硬关卡）

**准入**：Stage 4 准出 + Stage 1 PoC-B 通过。

**目标**：锁屏唤醒可用 + 内存可控 + 不被杀。

**交付物**：
- [ ] Phantom Mic 真机验证（依赖 PoC-B 通过；LSPosed Vector 框架已激活是前提）
- [ ] appops 自动授权（KSU post-fs-data.sh）
- [ ] **9 步 MIUI 白名单自动化检测**（[E13](#15-第二轮--第三轮-web-核实补正已修复)，新增"应用智能休眠"检测）
- [ ] 内存监控（lmkd 阈值告警）
- [ ] 真机连续 24 小时稳定性测试
- [ ] 发热监控 + 降频策略
- [ ] 诊断脚本 [07_troubleshooting.md §8.1](./07_troubleshooting.md)

**准出条件**（硬关卡，[E10](#15-第二轮--第三轮-web-核实补正已修复) 修订）：
- 锁屏 1 小时后唤醒率 > 70%
- 24 小时无崩溃重启
- **1.5B 串行峰值内存 < 7.6GB**（[D30](../DECISIONS.md)；3B 8GB 设备不可用）
- 24h 无 OOM kill（即使峰值 7.89GB 最坏叠加场景也需在 lmkd 阈值之上存活）
- 8+ Gen 1 满载温度 < 75℃

**回退路径**：Phantom Mic 不通过 → 仅亮屏可用 + appops 兜底；温度过高 → 限制每日对话时长。

### Stage 6：Phase 5 发布（预计 1-2 周）

**准入**：Stage 5 硬关卡通过 + **TTS/KWS 模型训练数据许可遗留已解决**（[D22](../DECISIONS.md) + [D23](../DECISIONS.md)）。

**交付物**：
- [ ] CI/CD（GitHub Actions：APK 自动构建 + KSU zip 打包）
- [ ] Release v0.1.0：APK + KSU zip + Phantom Mic + **LSPosed Vector v2.0** 安装指引（第六轮 R5 订正版本号）
- [ ] 性能基准 [08_performance_bench.md](./08_performance_bench.md)
- [ ] 用户文档（[05_deploy_guide.md](./05_deploy_guide.md) 已就绪，含 9 步白名单 + HyperOS 3 前置）
- [ ] Issue 模板 + 用户反馈通道
- [ ] **TTS/KWS 训练数据许可确认**（Phase 5 发布前必做）：
  - TTS：vits-zh-ll 训练数据来源确认 / 替换为 Apache 2.0 数据自训 / 寻找明确许可的中文 TTS
  - KWS：WenetSpeech 后续许可更新确认 / 替换为 gigaspeech KWS（英文）

**准出条件**：
- 至少 1 名外部用户（非开发者）按文档独立部署成功
- 性能基准数据完整（tok/s / RTF / 内存 / 温度 / 唤醒率）
- **TTS + KWS 模型训练数据许可合规**（无 NON-COMMERCIAL 遗留）

**回退路径**：外部用户反馈严重问题 → 回到 Stage 5 修复；TTS/KWS 许可未解决 → 推迟发布 / 改用其他模型。

### Stage 7-10：Phase 6-9 扩展功能（每 Phase 预计 2-4 周）

| Stage | Phase | 准入 | 核心交付 | 关键风险 |
|---|---|---|---|---|
| 7 | Phase 6 联网工具 | Stage 6 | 全局开关 + 4 工具 + **Qwen 1.5B function calling 用 32-token CoT prompt**（[R2](#14-第一轮风险升级与新发现已修复)） | 1.5B FC 准确率 44%→64%，复杂工具不可靠 |
| 8 | Phase 7 手机控制 | Stage 7 | 4 工具（通知朗读 + R10/R11 敏感过滤 / 应用启动 monkey / 系统设置 / 截屏看屏）；微信回复降级为"只朗读不回复"（[D29](../DECISIONS.md)） | A2A 不对第三方 APK 开放；通知使用权需用户手动授权 |
| 9 | Phase 8 平台化 | Stage 8 | **Ktor 内嵌 HTTP server**（[D28](../DECISIONS.md)）+ Widget + MCP server（modelcontextprotocol/kotlin-sdk + kaeawc/android-mcp-sdk 参考，[E8](#15-第二轮--第三轮-web-核实补正已修复)） | MCP 协议版本演进需 Phase 8 实施前复查 |
| 10 | Phase 9 多模态 | Stage 9 | ImageChatTool + IMAGE_CAPTURING 状态机 + GLM-4V（**需实名认证 + 流量警示**，[R12](#152-第三轮新增风险r10-r13) + [R13](#152-第三轮新增风险r10-r13)） | 多模态 API 需联网 + 联网开关联动 + 实名认证门槛 |

### Stage 11：Phase 10 UX 增强（预计 3-4 周）

**准入**：Stage 10 准出。

**交付物**（按优先级）：
- [ ] P0: G17 全局暂停（Widget + 通知按钮，先不做双击电源键 LSPosed 注入）
- [ ] P0: G9 音频路由（跟随系统 + 蓝牙测试）
- [ ] P1: G1 多用户（profile 切换 + Room 表）
- [ ] P1: G2 儿童模式（双层过滤 + 时长限制）
- [ ] P2: G6 a11y（TalkBack + 字幕 + 状态颜色）
- [ ] P2: G5 i18n（中英双语 MVP）

**准出条件**：每个 G 项的验收标准（见 [13_phase10_ux_enhancements_design.md §0](./13_phase10_ux_enhancements_design.md)）。

---

## 三、风险关卡总览

| 关卡 | 位置 | 不通过则 |
|---|---|---|
| Gate 0 | Stage 0 完成 | 文档可信度崩塌，不启动任何代码 |
| Gate 1 | Stage 1 PoC-A/B/C | 退到 fallback 路径，可能牺牲锁屏唤醒 |
| Gate 2 | Stage 5 真机 24h 稳定性 | 不发布 v0.1.0 |
| Gate 3 | Stage 6 发布前 TTS/KWS 许可遗留 | 推迟发布 / 改用其他模型 |
| Gate 4 | Stage 7 Phase 6 工具调用 | 1.5B FC 不够 → 切 3B 或加 CoT prompt |
| Gate 5 | Stage 8 Phase 7 通知朗读 | A2A 不开放 + 通知使用权需手动授权 + R10/R11 敏感过滤 |

---

## 四、决策修订清单（三轮累计）

基于三轮复审结果，对 [DECISIONS.md](../DECISIONS.md) 的修订清单：

### 第一轮修订（已完成）

| 决策 | 修订内容 |
|---|---|
| D7 | 修正 llama.android 路径为 `examples/llama.android`；ToolNeuron 参考文件改为 `InferenceService.kt` + `InferenceClient.kt`（非 LlamaEngine.kt）；libllama.so 体积 ~50MB→~30MB |
| D22 | vits-zh-ll 改为"社区贡献，许可未明确声明"；备选升级为 `matcha-icefall-zh-baker`；ASR 文件名补 `-2023-02-20` 后缀 |
| D23 | KWS 模型从虚构的 `zh-vgg` 改为 `kws-zipformer-wenetspeech-3.3M-2024-01-01`（中文默认）；补记 text2token 自定义词流程 |
| 新增 D24 | Qwen 商标限制：不写 "Powered by Qwen"，改 "使用 Qwen2.5 模型" |
| 新增 D25 | Qwen2.5-1.5B function calling 限制：Phase 6 工具调用需 32-token CoT prompt 或换 3B |
| 新增 D26 | TTS 无 chunk 回调：MVP 用 "先合成完整 PCM 再播"，Phase 10 G6 字幕按 200ms 切 PCM |
| 新增 D27 | Direct Boot + sherpa-onnx 组合未公开验证，Stage 1 PoC-A 为硬关卡 |

### 第二轮 + 第三轮修订（本轮已完成）

| 决策 | 修订内容 | 关联发现 |
|---|---|---|
| D6 修订 | HyperOS 2 → HyperOS 3（OS3.0.1.0.VLFCNXM 2026-01-20 OTA / OS3.0.2.0.VLFCNXM 2026-04-13 Fastboot） | F2 |
| D9 修订 | LSPosed → LSPosed Vector v2.0.3-7716（v2.0.1 2026-04-21 / v2.0.2 2026-05-05 / v2.0.3-7716 2026-05-20，支持 Android 8.1-17 Beta 3） | F1 |
| D14 修订 | Phantom Mic 兼容性 HyperOS 2 → HyperOS 3；补"LSPosed Vector 框架升级 ≠ Phantom Mic 模块兼容性升级" | F1 + F2 |
| D22 修订 | matcha-icefall-zh-baker 标为 NON-COMMERCIAL（训练数据 Data-Baker NON-COMMERCIAL ONLY） | E11 |
| D23 修订 | kws-zipformer-wenetspeech 标为 NON-COMMERCIAL（训练数据 WenetSpeech 论文原文 CC-BY 4.0 non-commercial）；MVP 改用 gigaspeech KWS 作为备选 | E12 |
| 新增 D28 | HTTP server 选型：Ktor 替代 NanoHTTPD（NanoHTTPD last commit 2 年前，仅 HTTP 1.1，无内置 SSE） | E7 |
| 新增 D29 | 微信 A2A 协议不对第三方 APK 开放；Phase 7 微信回复降级为"通知朗读 + 引导手动回复" | E9 |
| 新增 D30 | 3B 模型在 8GB RAM 设备上必 OOM（峰值 8.59GB > 8GB 物理上限），不可用 | E10 |
| 新增 D31 | HyperOS 2.0+ 应用智能休眠必关（白名单不豁免），部署白名单 8 步 → 9 步 | E13 |

---

## 五、立即行动项

按优先级排序，已完成的全部修复项：

1. ✅ **Stage 0 全部修订**（第一轮 + 第二轮 + 第三轮，共 9 + 13 = 22 项）
2. ✅ **Stage 1 PoC 计划文档化**（PoC-A/B/C 的具体测试用例，PoC-B 增加 LSPosed Vector 框架激活验证）
3. ✅ **DECISIONS.md 新增 D24-D31**（共 8 个新决策，覆盖三轮全部发现）
4. ✅ **各文档交叉修订**（搜索幻觉引用 + 训练数据许可 + HyperOS 3 + LSPosed Vector）

**下一阶段应启动**：
- Stage 1 三个 PoC 的最小测试代码骨架（PoC-A/B/C）
- Stage 2 Phase 1 MVP 工程结构搭建（参考 ToolNeuron + llama.android）

---

## 六、关键来源（三轮累计）

### 第一轮来源
- [llama.cpp b9830 release](https://github.com/ggml-org/llama.cpp/releases/tag/b9830)
- [llama.android 官方模块](https://github.com/ggml-org/llama.cpp/tree/master/examples/llama.android)（**注意路径**）
- [llama.cpp docs/android.md](https://github.com/ggml-org/llama.cpp/blob/master/docs/android.md)
- [sherpa-onnx v1.13.3 release](https://github.com/k2-fsa/sherpa-onnx/releases/tag/v1.13.3)
- [sherpa-onnx KWS 预训练模型清单](https://k2-fsa.github.io/sherpa/onnx/kws/pretrained_models/index.html)
- [sherpa-onnx VAD 文档](https://k2-fsa.github.io/sherpa/onnx/vad/silero-vad.html)
- [sherpa-onnx Kotlin API Examples](https://k2-fsa.github.io/sherpa/onnx/kotlin-api/examples.html)
- [voiceping 离线 ASR 基准](https://voiceping.net/ja/blog/research-offline-speech-transcription-benchmark/)
- [voiceping 离线 TTS 基准](https://voiceping.net/ja/blog/research-offline-tts-eval/)
- [Qwen License 商用指南](https://m.php.cn/faq/2496984.html)
- [Qwen2.5-1.5B Function Calling BFCL v3 论文](https://arxiv.org/pdf/2604.02155)
- [Phantom Mic 上游仓库](https://github.com/Xposed-Modules-Repo/tn.amin.phantom_mic)
- [Android Direct Boot 官方文档](https://developer.android.google.cn/privacy-and-security/direct-boot)
- [BCR Direct Boot 参考实现](https://blog.csdn.net/gitblog_00991/article/details/156410825)
- [ToolNeuron 仓库](https://github.com/Siddhesh2377/ToolNeuron)（429★，MIT，参考 `service/inference/` 目录）

### 第二轮 + 第三轮 web 核实来源

- **[F1] LSPosed Vector**：v2.0.1 2026-04-21 / v2.0.2 2026-05-05 / **v2.0.3-7716 2026-05-20**，支持 Android 8.1-17 Beta 3（fork 自 LSPosed 原项目，社区接力维护）
- **[F2] HyperOS 3 K50U**：国行 OS3.0.1.0.VLFCNXM 2026-01-20 OTA / OS3.0.2.0.VLFCNXM 2026-04-13 Fastboot，基于 Android 15
- **[E7] NanoHTTPD**：last commit ~2 年前，仅 HTTP 1.1，无内置 SSE，162 open issues 无人响应（open-awesome.com 实测）
- **[E8] MCP Kotlin SDK**：[github.com/modelcontextprotocol/kotlin-sdk](https://github.com/modelcontextprotocol/kotlin-sdk)，modelcontextprotocol 官方组织，1393★，0.13.0；Android PoC：[github.com/kaeawc/android-mcp-sdk](https://github.com/kaeawc/android-mcp-sdk)
- **[E9] 微信 A2A**：2026-06-04 与华为/小米/荣耀/OPPO/vivo 合作推出 A2A 助手能力，仅对系统级 AI 助手开放，第三方 APK 不可接入（[hellochinatech.com](https://hellochinatech.com/p/wechat-ai-gatekeeper) / [itbear.com](https://www.itbear.com/technews/wechat-teams-up-with-phone-makers-to-let-ai-assistants-send-messages-and-start-calls-by-voice/)）
- **[E11] Data-Baker**：TTS 训练数据，NON-COMMERCIAL ONLY
- **[E12] WenetSpeech**：训练数据论文原文明确 "licensed for non-commercial usage under CC-BY 4.0"
- **[E13] HyperOS 2.0+ 应用智能休眠**：新增的 AI 行为预测休眠机制，电池白名单不豁免，必须单独关闭
- **[R10-R13]**：第三轮深度审视发现，详见 §1.5.2

---

## 七、本轮（第三轮）修复完成清单

### 文档修订清单（11 份文件）

| 文件 | 修订项 | 状态 |
|---|---|---|
| [DECISIONS.md](../DECISIONS.md) | D6 HyperOS 3 + D9 LSPosed Vector + D14 HyperOS 3 兼容性 + D22 NON-COMMERCIAL + D23 NON-COMMERCIAL + 新增 D28-D31 | ✅ |
| [03_architecture_detail.md](./03_architecture_detail.md) | §6 内存算法修正（1.5B 串行峰值 ~7.5GB / 3B 8.59GB 必 OOM） | ✅ |
| [05_deploy_guide.md](./05_deploy_guide.md) | §0 前置条件 HyperOS 3 + LSPosed Vector；§5 9 步白名单（新增应用智能休眠） | ✅ |
| [06_lspoded_setup.md](./06_lspoded_setup.md) | §1 LSPosed Vector v2.0.3 + §6.1 HyperOS 3 兼容性 | ✅ |
| [10_phase7_phone_control_design.md](./10_phase7_phone_control_design.md) | §2.3.2 微信 A2A 移除；§2.4.1 R10/R11 敏感通知过滤 | ✅ |
| [11_phase8_platform_design.md](./11_phase8_platform_design.md) | §1.2/§1.3/§1.5 NanoHTTPD → Ktor；§3.1 MCP Kotlin SDK 归属修正 | ✅ |
| [12_phase9_multimodal_design.md](./12_phase9_multimodal_design.md) | §3.3.1 R13 base64 流量量化；§4.1 R12 GLM-4V 实名认证 | ✅ |
| [14_feasibility_recheck_and_plan.md](./14_feasibility_recheck_and_plan.md) | 本文档重写 | ✅ |

### 决策修订清单（11 个决策）

| 决策 | 类型 | 内容 |
|---|---|---|
| D6 | 修订 | HyperOS 2 → HyperOS 3 |
| D9 | 修订 | LSPosed → LSPosed Vector v2.0.3-7716 |
| D14 | 修订 | Phantom Mic 兼容性 HyperOS 2 → HyperOS 3 + LSPosed Vector 框架升级 |
| D22 | 修订 | matcha-icefall-zh-baker 标为 NON-COMMERCIAL |
| D23 | 修订 | kws-zipformer-wenetspeech 标为 NON-COMMERCIAL |
| D28 | 新增 | HTTP server 选型 NanoHTTPD → Ktor |
| D29 | 新增 | 微信 A2A 协议不对第三方 APK 开放 |
| D30 | 新增 | 3B 模型 8GB RAM 设备不可用 |
| D31 | 新增 | HyperOS 2.0+ 应用智能休眠必关 |

### 风险新增清单（4 项）

| 风险 ID | 描述 | 缓解 |
|---|---|---|
| R10 | 验证码短信被 TTS 朗读明文 | 默认识别验证码模式不朗读明文 + 锁屏不朗读敏感通知 |
| R11 | 银行/支付通知被朗读 | 默认对敏感关键词过滤，仅朗读 appName |
| R12 | GLM-4V 需实名认证 | UI 强制弹实名说明 + 备选 API |
| R13 | base64 流量未量化 | MVP 默认高压缩 + UI 流量警示 |

---

**第三轮复审完成日期**：2026-06-30
**下一阶段**：Stage 1 三个 PoC 测试代码骨架
