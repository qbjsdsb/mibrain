# 第二轮严谨审视：5 个新发现与修订

> 审视日期：2026-06-30
> 触发条件：用户要求"反复检查设计是否可用，借鉴的项目都不要忘"
> 结论：发现 5 个之前漏掉的问题，全部修订并并入设计
>
> **⚠️ 2026-06-30 二轮深度检查后的重大修订**（本审视记录保留作历史追溯）：
> - **发现 1 已被推翻**（[X7 废弃](../DECISIONS.md)）：llama-server HTTP 路径在 Android 上有根因问题（S1/S2/S3），切回 JNI（[D7 修订](../DECISIONS.md)）。ToolNeuron 重新评估为 Kotlin+Compose 全栈，真实推理封装为 `InferenceService.kt` + `InferenceClient.kt`（位于 `service/inference/` 目录）可作为参考样板（[X2 修正](../DECISIONS.md)）
> - **发现 2 已升级**：模型路径从"app 私有目录"进一步改为"DE 加密区 + Direct Boot"（[D21 新增](../DECISIONS.md)），解决 FBE 加密锁屏读不到模型
> - **发现 4 已升级**：Phantom Mic 风险从"中"升级为"高"（[D14](../DECISIONS.md)），v2.0 已近 2 年未更新
> - **发现 5 已升级**：唤醒词引擎从 openWakeWord 改为 sherpa-onnx KWS（[D23 新增](../DECISIONS.md)），弃用 openWakeWord
> - 默认模型从 3B 改为 1.5B（[D1 修订](../DECISIONS.md)），解决内存预算
> - ASR/TTS 模型换 Apache 2.0 许可（[D22 新增](../DECISIONS.md)），解决许可冲突
>
> 以下原文保留供追溯，注意每个发现的"修订"小节后都有"2026-06-30 二轮检查后"标注。

---

## 发现 1：fork ToolNeuron 的推理封装（原误称 LlamaEngine.kt，实为 InferenceService.kt + InferenceClient.kt）路径不成立

### 之前的判断
"直接 fork ToolNeuron 的推理封装（InferenceService.kt + InferenceClient.kt，MIT 许可），省 1500+ 行 Kotlin 代码量"

### 重新查证后发现
- ToolNeuron 用的是**它自己写的 llama.cpp Android binding**，位于 `app/src/main/cpp/`
- 这是 **C++ 代码 + JNI 包装**，不是纯 Kotlin
- ToolNeuron 的 Kotlin 端只是调用方，真正的工程量在 C++ 那层

### 修订
- 不再依赖 fork ToolNeuron 的 Kotlin 代码
- 改为**直接使用 llama.cpp 官方 `llama-server` 二进制**（已验证 android-arm64 包存在）
- APK 通过 HTTP 调用本地 `127.0.0.1:8080` 的 llama-server
- **彻底跳过 JNI 集成**，省掉 1500+ 行 Kotlin + C++ 工作量

### 新的架构调整
```
旧方案: APK → JNI → libllama.so → 模型
新方案: APK → HTTP → llama-server 二进制 → libllama.so → 模型
                      ↑ 由 KSU service.sh 启动
```

**优点**：
- 工程量降低 70%（不用写 JNI/C++）
- llama-server 自带 OpenAI 兼容 API + SSE 流式
- 升级 llama.cpp 只换二进制，不动 APK

**缺点**：
- 多一个本地 HTTP 进程（但只占 ~10MB 内存）
- 通信有 ~1ms 本地回环延迟（可忽略）

> **🔴 2026-06-30 二轮检查后推翻**（[X7 废弃](../DECISIONS.md) + [D7 修订](../DECISIONS.md)）：
> 深度检查 + 开源调研发现该 HTTP 路径在 Android 上有根因性问题：
> 1. **无任何先例**：所有主流 Android LLM 项目（PocketPal/ToolNeuron/Meta Llama Stack）均走 JNI
> 2. **S1**：root 启动的 llama-server 在 `su` SELinux 域访问 `/sdcard/` 模型需 sepolicy 放行 fuse 类型，未定义
> 3. **S2**：FBE 加密下锁屏首次解锁前 `/sdcard/` 是空 stub，service.sh 开机自启必然失败
> 4. **S3**：Android stock ROM 不带 `curl`，service.sh 健康检查失效，watchdog 误判反复重启
>
> 同时 ToolNeuron 重新评估为 **Kotlin + Compose 全栈同构**（不是原判断的"C++ + JNI 不是纯 Kotlin"），真实推理封装 `InferenceService.kt` + `InferenceClient.kt`（位于 `service/inference/` 目录）可作为参考样板（[X2 修正](../DECISIONS.md)）。
>
> **最终结论**：切回 JNI（[D7 修订](../DECISIONS.md)），参考 llama.android 官方模块 + ToolNeuron `InferenceService.kt` + `InferenceClient.kt`（位于 `service/inference/` 目录）实现。

---

## 发现 2：模型总体积超 2GB，必须改成"运行时下载"

### 之前的判断
"模型放 `/data/adb/mibrain/models/`"

### 重新计算
```
paraformer 流式 ASR:        ~250MB
VITS aishell3 中文 TTS:     ~150MB
openWakeWord hey_jarvis:    ~10MB
Qwen2.5-3B Q4_K_M:          ~2GB
─────────────────────────────────
合计:                       ~2.4GB
```

### 问题
- Google Play 单 APK 限制 200MB
- 即使自签名 APK，2.4GB 下载也难
- 模型升级时要重打 APK，不灵活

### 修订
模型**全部不打包**，改为：
1. APK 首次启动引导用户下载
2. 下载到 app 私有目录（无需跨 SELinux 域）
3. 提供两个下载源（HF + 阿里 OSS 镜像）兜底
4. 校验 SHA256，断点续传

KSU 模块**也不打包模型**，只放二进制和脚本。

> **⚠️ 2026-06-30 二轮检查后升级**（[D21 新增](../DECISIONS.md)）：
> 模型路径从"app 私有目录（CE 加密区）"进一步改为 **"DE 加密区 + Direct Boot"**：
> - 路径：`/data/user_de/0/com.mibrain/files/models/`
> - 解决 S2 FBE 加密问题：CE 区在用户首次解锁前不可读，DE 区在开机即可读
> - 解决 S1 跨 SELinux 域问题：DE 区属于 `app_data_file` 类型，JNI 调用不跨域
> - Direct Boot 让 Phase 4 "锁屏唤醒" 验收标准在加密设备上也能达成
>
> 同时模型清单更新（详见 [03_architecture_detail.md §1 关键文件路径](./03_architecture_detail.md)）：
> - 默认模型从 Qwen2.5-3B 改为 **Qwen2.5-1.5B Q4_K_M**（~1GB，[D1 修订](../DECISIONS.md)）
> - ASR 从 paraformer（CC BY-NC）改为 **sherpa-onnx-streaming-zipformer-bilingual-zh-en**（Apache 2.0，[D22](../DECISIONS.md)）
> - TTS 从 aishell3（CC BY-NC-ND）改为 **sherpa-onnx-vits-zh-ll**（社区贡献，许可未明确声明；HF 卡未声明许可，[D22](../DECISIONS.md)）
> - 唤醒词从 openWakeWord 改为 **sherpa-onnx KWS**（[D23 新增](../DECISIONS.md)）

---

## 发现 3：sherpa-onnx Kotlin API 是官方维护的（之前不确定）

### 之前的判断
"Sherpa-onnx 官方文档列了 Kotlin API 链接，但 Kotlin API 实际是社区维护还是官方？"

### 重新查证
- 官方仓库 `k2-fsa/sherpa-onnx` 里有专门的 `sherpa-onnx/kotlin-api/` 目录
- 是 **k2-fsa 官方维护**，不是社区
- 跟随版本发布，质量稳定

### 结论
**Kotlin API 完全可用**，无需自己写 wrapper。

### 文档
官方 Kotlin API 文档：https://k2-fsa.github.io/sherpa/onnx/kotlin-api/

> **✅ 2026-06-30 二轮检查后确认有效**：此发现未被推翻，sherpa-onnx v1.13.3 一个 AAR 同时提供 ASR/TTS/VAD/KWS 四能力，技术栈统一（[D8](../DECISIONS.md)、[D22](../DECISIONS.md)、[D23](../DECISIONS.md)）。

---

## 发现 4：Phantom Mic 在 Android 14+ 的兼容性未验证

### 之前的判断
"Phantom Mic v2.0 已经验证可达"

### 重新查证
- 红米 K50U 原厂 HyperOS 1 基于 Android 12
- 现役已升级到 **HyperOS 3 基于 Android 15**（[F2](./14_feasibility_recheck_and_plan.md) web 核实确认：OS3.0.1.0.VLFCNXM 2026-01-20 OTA / OS3.0.2.0.VLFCNXM 2026-04-13 Fastboot）
- Android 14+ 引入了 `foregroundServiceType="microphone"` 强制要求
- Phantom Mic 项目 README 未明确声明 Android 14/15 兼容

### 风险等级
🟡 中（可能能用，需要真机验证）

> **2026-06-30 更新 + 第四轮 web 核实确认**：风险升级为 🔴 高。Phantom Mic v2.0 自 2024-07 发布至本次设计冻结已近 2 年未更新，**第四轮 web 核实确认上游已停滞 23 个月**（自 2024-07-24 v2.0 后零更新），上游活跃度存疑，HyperOS 3（Android 15）兼容性更不确定。详见 [DECISIONS.md D14](../DECISIONS.md) 与 [01_feasibility_verification.md §三 风险 1](./01_feasibility_verification.md)。

### 修订
1. Phase 1-3 不依赖 Phantom Mic，先做"按按钮触发"的 APK
2. Phase 4 真机验证 Phantom Mic 是否需要更新版本或备选模块
3. 增加备选方案：`appops set <uid> RECORD_AUDIO allow`（root shell 命令）

> **✅ 2026-06-30 二轮检查后确认有效 + 第四轮确认上游停滞**：此发现升级后并入 [D14](../DECISIONS.md)。Phase 1 启动前去 [上游仓库](https://github.com/Xposed-Modules-Repo/tn.amin.phantom_mic/releases) 复查活跃度（第四轮已确认无新版本）+ Issues 区看 HyperOS 3 / Android 15 反馈；Phase 4 真机验证后准备 fallback；**第四轮补降级方案**（WhatsMicFix 改造 / 放弃锁屏唤醒，详见 D14）。

---

## 发现 5：唤醒词"嘿小脑"没有现成模型，需自训

### 之前的判断
"用 openWakeWord，唤醒词定什么默认'嘿小脑'"

### 重新查证
- openWakeWord 仓库提供的现成模型只有英文：`hey_jarvis.onnx`, `hey_mycroft.onnx` 等
- 中文唤醒词需要自己采集 100+ 句样本 + 用 ONNX 训练
- 工程量约 1-2 天（采集 + 标注 + 训练）

### 修订
1. **Phase 3 唤醒词阶段**才做中文唤醒词训练
2. **MVP（Phase 1-2）**先用英文 `hey_jarvis` 跑通链路
3. 或者**完全跳过唤醒词**，用桌面快捷方式/双击电源键触发，到 Phase 3 再升级

### 备选：sherpa-onnx KWS 中文模型
- 已验证 `sherpa-onnx-kws-zipformer-wenetspeech-3.3M-2024-01-01.tar.bz2` 可下载（原引用 `zh-vgg` 不存在，真实模型为 zipformer 架构）
- 但默认关键词是预设的，自定义仍需训练

> **⚠️ 2026-06-30 二轮检查后升级**（[D23 新增](../DECISIONS.md)）：
> 唤醒词引擎从 openWakeWord 改为 **sherpa-onnx KWS**（keyword spotting）：
> 1. sherpa-onnx v1.13.3 已内置 KWS 能力（含 zipformer KWS 模型，原引用 `zh-vgg` 不存在，真实模型为 zipformer 架构），无需另装 openWakeWord + 自写 Kotlin wrapper
> 2. 统一技术栈：sherpa-onnx 一个 AAR 同时提供 ASR + TTS + VAD + KWS 四件套，减少依赖
> 3. openWakeWord 自写 wrapper 与 sherpa-onnx 选型重复
>
> MVP 仍用英文 `hey_jarvis`（[D2](../DECISIONS.md)），Phase 3 自训中文"嘿小脑"。原"openWakeWord 没有现成中文模型"的具体理由随 [D23](../DECISIONS.md) 失效，但 MVP 先英文的结论不变。

---

## 汇总：对原设计的修订

| 原设计 | 修订为 | 修订理由 |
|---|---|---|
| APK 内嵌 llama.cpp JNI | 改用 HTTP 调用 llama-server 二进制 | 发现 1：JNI 工程量太大 |
| 模型放 `/data/adb/mibrain/models/` | 改为运行时下载到 app 私有目录 | 发现 2：跨 SELinux 域访问麻烦 |
| sherpa-onnx Kotlin API 待确认 | 确认是官方维护，直接用 | 发现 3：调研完成 |
| Phantom Mic 解决一切录音问题 | 改为 Phase 4 才验证 | 发现 4：兼容性未验证 |
| 唤醒词用"嘿小脑" | MVP 用 hey_jarvis 或手动触发 | 发现 5：中文需自训模型 |

> **⚠️ 2026-06-30 二轮检查后的最终修订表**：
>
> | 本审视的修订 | 二轮检查后的最终状态 | 决策编号 |
> |---|---|---|
> | APK 内嵌 JNI → HTTP llama-server | ❌ 推翻，切回 JNI（参考 llama.android + ToolNeuron） | [D7 修订](../DECISIONS.md)、[X7](../DECISIONS.md) |
> | 模型放 app 私有目录 | ⚠️ 升级为 DE 加密区 + Direct Boot | [D21 新增](../DECISIONS.md) |
> | sherpa-onnx Kotlin API 官方 | ✅ 确认有效，并扩展为全栈统一（ASR/TTS/VAD/KWS） | [D8](../DECISIONS.md)、[D22](../DECISIONS.md)、[D23](../DECISIONS.md) |
> | Phantom Mic 风险中 | 🔴 升级为高（v2.0 近 2 年未更新） | [D14](../DECISIONS.md) |
> | 唤醒词用 openWakeWord | ⚠️ 改为 sherpa-onnx KWS，弃用 openWakeWord | [D23 新增](../DECISIONS.md) |
> | 默认模型 Qwen2.5-3B | ⚠️ 降级为 1.5B（解决内存预算） | [D1 修订](../DECISIONS.md) |
> | ASR/TTS 模型许可 | ⚠️ 换 Apache 2.0 许可的 sherpa-onnx 官方模型 | [D22 新增](../DECISIONS.md) |

## 借鉴清单（再次确认不遗漏）

| 项目 | 用途 | 借鉴方式 |
|---|---|---|
| ToolNeuron | Kotlin APK 整体架构参考 | 不 fork 代码，仅参考结构 |
| WhatsMicFix-LSPosed | LSPosed 双 scope 思路 | 思路借鉴，不直接用代码 |
| HostAI | LiteRT-LM 在 Android 上的可行性证据 | 不用代码 |
| tasker-mcp | Go 在 Android 部署模式 | 已废弃（不再用 Go daemon） |
| Phantom Mic | 后台录音 hook | 用户自行安装，不打包 |
| llama.cpp | LLM 推理 | 直接用二进制 |
| sherpa-onnx | ASR/TTS/VAD/KWS | 直接用 AAR |

> **⚠️ 2026-06-30 二轮检查后更新借鉴清单**：
> - ToolNeuron 从"仅参考结构"升级为"参考结构 + `InferenceService.kt` + `InferenceClient.kt` JNI 范式"（[X2 修正](../DECISIONS.md)）
> - llama.cpp 从"直接用二进制（llama-server HTTP）"改回"JNI 调用 libllama.so"（[D7 修订](../DECISIONS.md)）
> - 唤醒词从 openWakeWord 改为 sherpa-onnx KWS，openWakeWord 不再列入借鉴清单（[D23](../DECISIONS.md)）
> - 完整最新借鉴清单见 [00_design_overview.md §3](./00_design_overview.md)

---

## 结论

第二轮审视发现 5 个真实问题，全部已修订并并入设计。**整体方案可行性不下降，工程量反而降低**（跳过 JNI 集成）。

可以进入 Phase 1。

> **⚠️ 2026-06-30 二轮检查后的最终结论**：
> 本审视的"跳过 JNI 集成"结论被推翻（[X7 废弃](../DECISIONS.md)），切回 JNI。但本审视提出的其他 4 个问题（模型路径、sherpa-onnx 官方 API、Phantom Mic 风险、唤醒词）经升级后仍有效。
>
> 二轮深度检查后共修订 7 项决策（D1/D7/D14/D21/D22/D23/X2），新增 4 项（D21/D22/D23/X7），整体方案可行性进一步提升。详见 [DECISIONS.md](../DECISIONS.md)。
>
> 可以进入 Phase 1（**注意 Phase 1 启动前先做 [D14](../DECISIONS.md) 的 Phantom Mic 上游活跃度复查**）。
