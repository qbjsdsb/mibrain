# MiBrain - 红米 K50U 本地 AI 助手 项目设计文档

> 本文档**仅做设计，不交付代码**。先想清楚再动手。

---

## 0. 项目元信息

| 项 | 值 |
|---|---|
| 项目名 | MiBrain（已确认，见 [D4](../DECISIONS.md)） |
| 目标设备 | 红米 K50 Ultra（骁龙 8+ Gen 1，8GB RAM，Adreno 730，HyperOS） |
| Root 方案 | KernelSU + ZygiskNext + LSPosed |
| 用户硬约束 | 只能保证 KSU + ZygiskNext + LSP，其他能力不保证（不自编译内核、不改 ROM、不长期维护复杂 hook） |
| 核心目标 | 离线、本地、隐私不出手机的语音助手 |
| 开发语言 | Kotlin（APK）+ Shell（KSU 脚本）+ 现成 arm64 二进制 |
| 当前阶段 | Phase 0：设计冻结（不交付代码，见 [README.md](../README.md)） |

---

## 1. 为什么要做这个项目（用户视角的问题陈述）

### 1.1 痛点

1. **商业助手体验差**：小爱、Siri 在国内受限于云识别 + 隐私顾虑
2. **本地 LLM 没有易用入口**：HostAI、Termux+llama.cpp 都需要懂技术
3. **MIUI/HyperOS 后台录音限制狠**：通用 APK 装上锁屏就哑
4. **现有方案要么太重要么太碎**：ToolNeuron 全功能但重；自己拼 HostAI+Tasker+PhantomMic 太碎没人维护

### 1.2 目标用户体验

> "我说『小脑』，手机在 3 秒内开始回答我的问题，全程离线。"

具体可量化的验收标准：

| 指标 | 目标 | 备注 |
|---|---|---|
| 唤醒响应延迟 | < 1s | 唤醒词检出 → 开始录音 |
| 录音 → 文本 | < 3s | 5 秒以内的短句 |
| 模型首 token | < 2s | 模型已 keep-alive 状态 |
| 首句语音回应 | < 5s | 含 TTS 合成 |
| 锁屏后可用率 | > 90% | 24 小时内随机测试 |
| 内存峰值 | < 6.5GB | 保证系统不卡 |
| 隐私 | 100% 离线 | 任何数据不上云 |

---

## 2. 与现有开源项目的差异化定位

### 2.1 调研的参考项目

| 项目 | 仓库 | 借鉴价值 | 不直接用的原因 |
|---|---|---|---|
| **ToolNeuron** | Siddhesh2377/ToolNeuron (258★, 927 commits) | 完整 Kotlin 架构、GGUF 加载、TTS/STT/RAG 全套 | 太重，927 commits 学习成本高；未针对 MIUI 优化；无 LSPosed hook |
| **HostAI** | wannaphong/android-hostai (v1.0.4, alpha) | LiteRT-LM + GPU 加速 + OpenAI API | 只解决"模型服务"一层，无语音、无 hook |
| **Phantom Mic** | Xposed-Modules-Repo/tn.amin.phantom_mic (LSPosed 官方) | native hook AudioRecord.cpp，成熟 | 只 hook 不提供调度，用户自行安装 |
| **WhatsMicFix** | D4vRAM369/WhatsMicFix-LSPosed (75 commits) | LSPosed 双 scope 架构（app+system） | 专为 WhatsApp，需改造 |
| **XAudioCapture** | wzhy90/XAudioCapture | hook `PRIVATE_FLAG_ALLOW_AUDIO_PLAYBACK_CAPTURE` | 只解决"播放录音"非"麦克风录音" |
| **tasker-mcp** | cj-elevate/tasker-mcp | Go 交叉编译 arm64、Tasker HTTP 协议 | 是 MCP server，方向不同 |
| **Hotword Plugin + Snowboy** | jolanrensen | Tasker 唤醒词插件 + 自训 .umdl 模型 | Snowboy 已半弃维 |
| **wyoming-satellite-termux** | pantherale0 | Termux 跑本地唤醒 | 依赖 Termux（用户已排除） |

### 2.2 我们的差异化

**MiBrain 的"窄而深"定位**：

- ❌ 不做：图像生成、RAG 知识库、插件系统（这些 ToolNeuron 已做得很好，用户可以直接装 ToolNeuron 补这些）
- ✅ 专注做：**MIUI/HyperOS 上的稳定后台语音对话**，这是 ToolNeuron 没解决的真问题
- ✅ 交付物：一个 KSU 模块 + 一个极简 APK + 配套 LSPosed 配置 + 部署脚本

**用一句话定义**：
> MiBrain 是一个针对小米 HyperOS 的"本地 LLM 语音助手"开箱即用方案，把"模型服务 + 语音 IO + 后台保活"三件事打包到一个 KSU 模块里，刷入即用。

---

## 3. 借鉴清单（具体到模块）

| MiBrain 模块 | 借鉴自 | 借鉴点 | 是否直接 fork |
|---|---|---|---|
| APK 主体 | ToolNeuron (re-write 分支) | 项目骨架、Gradle 配置、SAF 文件选择 | 否，参考重写 |
| LLM 推理 | llama.cpp 官方 llama-server 二进制 | OpenAI 兼容 HTTP API + SSE 流式 | 直接用二进制，APK 走 HTTP |
| TTS | ToolNeuron 的 sherpa-onnx 集成 | 10 声音、5 语言、onnx 模型加载 | 参考 |
| STT | ToolNeuron 的 sherpa-onnx ASR | 在线流式识别 + VAD | 参考 |
| 唤醒词 | Hotword Plugin 思路 | 持续监听 + 模型可换 | 重写为 Kotlin |
| LSPosed 后台录音 | Phantom Mic (现成模块) | native 层 AudioRecord.cpp hook | 装现成，不自写（[D9](../DECISIONS.md)） |
| KSU 模块壳 | 标准模板 | service.sh / post-fs-data.sh / sepolicy.rule | 直接用 |

> **废弃路径**（详见 [DECISIONS.md X1-X5](../DECISIONS.md)）：
> - ~~Go daemon~~（CGO + NDK 交叉编译门槛高）
> - ~~fork ToolNeuron LlamaEngine.kt~~（实际是 C++ + JNI，工程量并未省）
> - ~~自写 LSPosed hook~~（Phantom Mic 已完整覆盖 native hook）
> - ~~WhatsMicFix 改造~~（专为 WhatsApp，Phantom Mic 已够用）

---

## 4. 关键设计决策（带依据）

### 4.1 决策：用 Kotlin APK 而非 Go daemon（推翻之前的方案）

**原因**：
- 之前选 Go daemon，调研发现 Android 上 `CGO_ENABLED=0` DNS 会崩，必须 NDK 交叉编译，对用户门槛太高
- ToolNeuron 用 Kotlin + llama.cpp JNI 已经验证可行，927 commits 跑通
- 单 APK 比"APK + Go daemon"两层简单
- 8GB 内存下多一个 Go daemon 进程是负担

**结果**：所有逻辑都在 Kotlin APK 里，KSU 模块只负责"放二进制 + 改系统设置"

### 4.2 决策：llama.cpp 而非 LiteRT-LM

**原因**：
- LiteRT-LM（HostAI 用的）只支持 .litertlm 格式，模型选择面窄
- llama.cpp 支持 GGUF，社区模型海量（Qwen、Phi、Gemma 全有）
- llama.cpp 官方提供 android-arm64 预编译 `llama-server` 二进制，已验证可用（[01_feasibility_verification.md](./01_feasibility_verification.md) §2.1）
- GPU 加速：Adreno 730 支持 Vulkan，未来可启用 `-DLLAMA_VULKAN=ON`；OpenCL + Adreno 后端也有预编译包

**结果**：KSU 模块内放 `llama-server` 二进制（由 service.sh 启动），APK 通过 HTTP 调用，**不内嵌 .so 也不写 JNI**（详见 [D7](../DECISIONS.md) 与 [02_second_review.md](./02_second_review.md) 发现 1）

### 4.3 决策：sherpa-onnx 做 TTS + STT + 唤醒

**原因**：
- ToolNeuron 用 sherpa-onnx 同时做 TTS 和 STT，验证可行
- sherpa-onnx 支持流式 ASR（实时识别）+ 流式 TTS（边生成边播）
- 内置 VAD，省一层依赖
- 支持 openWakeWord 模型做唤醒词检测

**结果**：一套 sherpa-onnx 解决三件事，体积小、依赖少

### 4.4 决策：LSPosed 用 Phantom Mic 的 native hook

**原因**：
- MIUI 后台录音限制在 Java 层 hook 不够，必须 native
- Phantom Mic 已 hook `AudioRecord.cpp` native 层，覆盖 95% App
- 7.6MB APK，来自 LSPosed 官方模块仓库
- 我们不需要重写 hook，只要在 LSPosed 配置里启用即可

**结果**：不自己写 LSPosed 模块，**装现成的 Phantom Mic**，只在文档里给配置指引

> ⚠️ **风险升级（见 [D14](../DECISIONS.md)）**：原稿写"活跃维护（2024-07 v2.0）"，实际至 2026-06-30 已近 2 年未更新，上游可能停滞。进入 Phase 1 前先复查上游活跃度。

### 4.5 决策：不依赖 Tasker，APK 自带语音对话

**原因**：
- Tasker + AutoVoice 在国内依赖 Google 服务，不稳
- Tasker HTTP Request 不支持 SSE 流式
- 用户已说"APK 部分要完整模板"，那就一次性做到位

**结果**：APK 自己实现录音 → 唤醒 → ASR → LLM → TTS → 播放完整链路

---

## 5. 整体架构

```
┌────────────────────────────────────────────────────────────────────┐
│  Android 系统层（HyperOS）                                          │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ LSPosed 注入                                                  │   │
│  │  └─ Phantom Mic (现成) → hook AudioRecord.cpp native 层      │   │
│  │     → 让 APK 即使在后台也能持续拿到麦克风数据                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              ↑                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ MiBrain APK (Kotlin)                                          │   │
│  │  ┌────────────────────────────────────────────────────────┐  │   │
│  │  │ 前台服务 ForegroundService (microphone type)            │  │   │
│  │  │  ├─ AudioRecord 16kHz mono int16 (持续)                │  │   │
│  │  │  ├─ sherpa-onnx VAD (实时检测说话段)                    │  │   │
│  │  │  ├─ sherpa-onnx 唤醒词 (openWakeWord 模型)              │  │   │
│  │  │  └─ 检测到唤醒 → 切换到对话模式                          │  │   │
│  │  └────────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────────┐  │   │
│  │  │ 对话引擎 ConversationEngine                            │  │   │
│  │  │  ├─ sherpa-onnx 流式 ASR (说话→文本)                   │  │   │
│  │  │  ├─ HTTP 客户端 → 调用本地 llama-server (流式 SSE)       │  │   │
│  │  │  ├─ sherpa-onnx 流式 TTS (文本→语音，按句播放)         │  │   │
│  │  │  └─ AudioTrack 播放队列                                 │  │   │
│  │  └────────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────────┐  │   │
│  │  │ UI 层 (Jetpack Compose)                                │  │   │
│  │  │  ├─ 主界面：状态显示、对话气泡                          │  │   │
│  │  │  ├─ 设置：模型选择、唤醒词、TTS 声音、温度等           │  │   │
│  │  │  └─ 模型管理：下载、删除、激活                         │  │   │
│  │  └────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              ↑                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ KernelSU 模块 mi_brain_module                                │   │
│  │  ├─ service.sh：开机自启 llama-server + watchdog             │   │
│  │  │   └─ nohup llama-server --host 127.0.0.1 --port 8080 &    │   │
│  │  ├─ post-fs-data.sh：                                        │   │
│  │  │   ├─ 创建 /data/adb/mibrain/ 目录                         │   │
│  │  │   ├─ chcon SELinux 上下文                                 │   │
│  │  │   └─ appops set <uid> RECORD_AUDIO allow                 │   │
│  │  ├─ sepolicy.rule：放行 llama-server 网络与文件访问           │   │
│  │  ├─ system/etc/sysconfig/mibrain.xml：电池白名单             │   │
│  │  └─ libs/llama-server：llama.cpp android-arm64 预编译二进制   │   │
│  └─────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────┘
                              ↑
                              │ 用户手动放入 /sdcard/MiBrain/models/
                              │ （或 APK 内置下载器引导，见 §11 路线图 Phase 1）
                              │
┌────────────────────────────────────────────────────────────────────┐
│ 模型资产（不打包进模块与 APK，运行时下载到 app 私有目录）             │
│  ├─ qwen2.5-3b-instruct-q4_k_m.gguf   (~2GB，对话)                 │
│  ├─ sherpa-onnx-paraformer-zh (流式 ASR，~250MB)                  │
│  ├─ sherpa-onnx-vits-zh (TTS，~150MB)                              │
│  └─ openwakeword.onnx (唤醒词，~10MB)                              │
└────────────────────────────────────────────────────────────────────┘
```

> **架构变更说明（见 [02_second_review.md](./02_second_review.md) 发现 1 + 发现 2）**：
> - 推理后端从最初的 "APK 内嵌 llama.cpp JNI" 改为 "KSU 启动 llama-server 二进制 + APK 通过 HTTP 调用"，省掉 1500+ 行 JNI/C++ 工作量。
> - 模型不再放 `/data/adb/mibrain/models/`，改为运行时下载到 app 私有目录，避免跨 SELinux 域访问。
> - 详见 [DECISIONS.md D7](../DECISIONS.md)。

---

## 6. 组件选型与版本

| 组件 | 选型 | 版本 | 来源 |
|---|---|---|---|
| LLM 推理 | llama.cpp + llama-server 二进制 | b9844 | github.com/ggml-org/llama.cpp |
| Android arm64 二进制 | llama-b9844-bin-android-arm64 | b9844 | 同上 release（KSU 模块内放，APK 不内嵌） |
| 流式 ASR | sherpa-onnx (官方 Kotlin API) | v1.13.3 | github.com/k2-fsa/sherpa-onnx |
| 流式 TTS | sherpa-onnx VITS | v1.13.3 | 同上 |
| 唤醒词 | openWakeWord via sherpa-onnx | 0.6+ | github.com/dscripka/openWakeWord |
| LSPosed hook | Phantom Mic | 2.0 | github.com/Xposed-Modules-Repo/tn.amin.phantom_mic |
| UI 框架 | Jetpack Compose | BOM 2024.06+ | Google |
| 异步 | Kotlin Coroutines | 1.8+ | JetBrains |
| 持久化 | Room | 2.6+ | AndroidX |
| KSU 模块壳 | 标准 KSU 模板 | - | kernelsu.org |
| Gradle | Kotlin DSL | 8.5+ | - |

---

## 7. 数据流时序（一次完整对话）

```
用户              APK 前台服务          sherpa-onnx         llama.cpp        AudioTrack
 │                   │                    │                  │                 │
 │ "嘿小脑"          │                    │                  │                 │
 ├──────────────────►│                    │                  │                 │
 │                   │ 唤醒词检出 (80ms)  │                  │                 │
 │                   │ ─────────────────►│                  │                 │
 │                   │ ◄─────────────────┤ detected=true    │                 │
 │                   │                    │                  │                 │
 │ 提示音"嗯？"       │                    │                  │                 │
 │ ◄─────────────────┤                    │                  │                 │
 │                   │ 切换到 ASR 模式     │                  │                 │
 │                   │ ─────────────────►│                  │                 │
 │ "明天会下雨吗"     │                    │                  │                 │
 ├──────────────────►│ AudioRecord        │                  │                 │
 │                   │ ─────────────────►│ VAD + ASR        │                 │
 │                   │ ◄─────────────────┤ "明天会下雨吗"   │                 │
 │                   │                    │                  │                 │
 │                   │ POST /chat (本地函数调用)              │                 │
 │                   │ ─────────────────────────────────────►│ llama_decode    │
 │                   │ ◄───token─────────────────────────────┤ "明天"          │
 │                   │ ─────────────────────────────────────►│ ...             │
 │                   │ ◄───token─────────────────────────────┤ "明天晴"        │
 │                   │ 切句检测：检测到"。"                    │                 │
 │                   │ ─────────────────►│ TTS "明天晴"     │                 │
 │                   │ ◄─────────────────┤ audio chunk       │                 │
 │ 听到"明天晴"       │ ─────────────────────────────────────────────────────►│ AudioTrack
 │ ◄─────────────────┤                    │                  │                 │
 │                   │ ... 继续流式 ...    │                  │                 │
```

**关键设计点**：
- **句子切片阈值**：检测 `[。！？!?\n]` 任一符号即切句送 TTS，不等待整段生成完
- **回环防护**：TTS 播放期间，唤醒词检测暂停 + VAD 输入直接丢弃
- **超时降级**：若 5s 无 token，先 TTS"让我想想..."安抚用户

---

## 8. 仓库结构设计

### 8.1 单仓库 vs 多仓库

**单仓库**：所有组件放一起，方便用户一个 clone 拿全部。

### 8.2 仓库目录结构

```
mibrain/                                    # 仓库根
├── README.md                              # 项目介绍 + 快速开始
├── LICENSE                                # Apache 2.0
├── CONTRIBUTING.md                        # 贡献指南
├── docs/                                  # 文档（本设计文档在此）
│   ├── 00_design_overview.md              # 本文档（完整设计）
│   ├── 01_feasibility_verification.md     # 可行性验证报告（10/10 依赖可达）
│   ├── 02_second_review.md                # 第二轮严谨审视（5 个新发现 + 修订）
│   ├── 03_architecture_detail.md          # 详细架构与时序图（进程拓扑、协议、状态机）
│   ├── 04_build_guide.md                  # 编译指南 ⏳ Phase 1
│   ├── 05_deploy_guide.md                 # 部署指南（用户向，含 8 步白名单）
│   ├── 06_lspoded_setup.md                # LSPosed 配置指南（Phantom Mic 详解）
│   ├── 07_troubleshooting.md              # 故障排查（7 大类故障 + 排查流程）
│   └── 08_performance_bench.md            # 性能基准 ⏳ Phase 5
│
├── app/                                   # MiBrain APK（Android Studio 项目）
│   ├── app/
│   │   ├── build.gradle.kts
│   │   ├── src/main/
│   │   │   ├── AndroidManifest.xml
│   │   │   ├── java/com/mibrain/
│   │   │   │   ├── MainActivity.kt         # Compose 入口
│   │   │   │   ├── service/
│   │   │   │   │   ├── BrainForegroundService.kt   # 前台服务
│   │   │   │   │   └── AudioPipeline.kt            # 录音→VAD→唤醒→ASR
│   │   │   │   ├── engine/
│   │   │   │   │   ├── LlamaHttpClient.kt          # 调用本地 llama-server (HTTP + SSE)
│   │   │   │   │   ├── SherpaAsrEngine.kt         # 流式 ASR
│   │   │   │   │   ├── SherpaTtsEngine.kt         # 流式 TTS
│   │   │   │   │   ├── WakeWordEngine.kt          # 唤醒词
│   │   │   │   │   └── ConversationEngine.kt      # 编排
│   │   │   │   ├── data/
│   │   │   │   │   ├── ModelManager.kt             # 模型管理（运行时下载到 app 私有目录）
│   │   │   │   │   ├── SettingsRepository.kt
│   │   │   │   │   └── db/                          # Room
│   │   │   │   ├── ui/
│   │   │   │   │   ├── theme/
│   │   │   │   │   ├── screens/                    # 各 Compose 屏幕
│   │   │   │   │   └── components/
│   │   │   │   └── util/
│   │   │   ├── assets/
│   │   │   │   └── default_prompts.json            # 默认 system prompt
│   │   │   └── jniLibs/arm64-v8a/                  # 仅 sherpa-onnx 的 .so（APK 不内嵌 libllama.so）
│   │   │       ├── libsherpa-onnx-jni.so            # sherpa-onnx 预编译
│   │   │       └── libonnxruntime.so
│   │   └── proguard-rules.pro
│   ├── settings.gradle.kts
│   └── gradle/libs.versions.toml                    # 版本目录
│
├── ksu_module/                            # KSU 模块源码
│   ├── module.prop
│   ├── install.sh
│   ├── post-fs-data.sh                   # 创建目录 + chcon SELinux + appops
│   ├── service.sh                         # 启动 llama-server 二进制 + watchdog
│   ├── uninstall.sh
│   ├── sepolicy.rule
│   ├── system/etc/sysconfig/mibrain.xml   # 电池白名单
│   ├── libs/llama-server                  # llama.cpp android-arm64 预编译二进制
│   └── build.sh                           # 打包 zip 脚本
│
├── scripts/                               # 辅助脚本
│   ├── download_models.sh                 # 一键下载模型（HF + 阿里 OSS 镜像兜底）
│   ├── download_binaries.sh               # 一键下载 llama-server 二进制 + sherpa-onnx AAR
│   ├── verify_assets.sh                   # 校验所有下载文件的 SHA256
│   ├── build_ksu_zip.sh                   # 打包 KSU 模块 zip
│   └── ci/                                # CI 脚本 ⏳ Phase 5
│       ├── build_apk.sh                   # CI 中构建 APK
│       └── verify_design.sh              # CI 中验证设计文档完整性
│
├── .github/                               # GitHub 配置
│   ├── ISSUE_TEMPLATE/                    # Issue 模板（bug / feature / test_report）
│   └── workflows/                         # GitHub Actions ⏳ Phase 5
│       ├── build_apk.yml                  # 自动构建 APK
│       ├── build_ksu_module.yml           # 自动打包 KSU 模块
│       └── release.yml                    # 发布 release
│
└── third_party/                           # 第三方资源引用清单
    ├── NOTICES.md                         # 所有引用的开源项目
    └── licenses/                          # 完整许可文本 ⏳ Phase 1 补充
```

---

## 9. 模块职责矩阵

| 模块 | 职责 | 失败时的降级 |
|---|---|---|
| BrainForegroundService | 持续运行、保活、协调各 engine | 重启自身（watchdog） |
| AudioPipeline | 录音 + VAD + 唤醒词 | 显示"麦克风异常" toast |
| WakeWordEngine | 检测唤醒词 | 切换到桌面快捷方式手动触发 |
| SherpaAsrEngine | 流式语音转文本 | 提示用户重说 |
| LlamaEngine | LLM 流式生成 | 提示"模型加载失败" |
| SherpaTtsEngine | 文本转语音 | 退回 Android 内置 TTS |
| ConversationEngine | 编排上述四者 | 单点失败即停止当前对话 |
| ModelManager | 模型文件管理 | 缺失模型时引导下载 |
| SettingsRepository | 配置持久化 | 用默认配置 |
| UI 层 | 用户交互 | - |

---

## 10. 风险与缓解

| 风险 | 概率 | 影响 | 缓解策略 |
|---|---|---|---|
| 8GB 内存峰值触发 lmkd | 高 | 高 | 模型 keep-alive 5min 自动卸载；监听 onTrimMemory 主动释放 |
| MIUI 锁屏杀后台 | 高 | 高 | Phantom Mic hook + sysconfig 白名单 + 前台服务通知 |
| sherpa-onnx native 库体积大 | 中 | 中 | 用 proguard 移除未用 ABI；只打包 arm64-v8a |
| 唤醒词误触发 | 中 | 低 | 双段确认（唤醒后说"嗯？"再听命令） |
| **Phantom Mic 上游停滞 / HyperOS 2 不兼容** | **高** | **高** | 见 [DECISIONS.md D14](../DECISIONS.md)；备选 appops + 双触发兜底；Phase 1 前先复查上游活跃度 |
| llama.cpp 升级破坏 llama-server API | 中 | 中 | 锁版本到 b9844；CI 自动验证 HTTP 协议 |
| 麦克风被其他 App 占用 | 低 | 中 | 监听 AudioFocus 变化；占用时静默等待 |

---

## 11. 交付路线图

### Phase 0：设计冻结（当前）
- ✅ 本文档完成
- ⏳ 等待用户审阅
- ⏳ 等待用户给 GitHub 信息

### Phase 1：MVP 单链路打通（最小可用）
- [ ] Kotlin APK 骨架 + Compose UI（参考 ToolNeuron）
- [ ] KSU 模块壳：service.sh 启动 llama-server 二进制 + watchdog
- [ ] APK 内 LlamaHttpClient.kt：调用本地 127.0.0.1:8080（非 JNI）
- [ ] 一个静态对话界面（按按钮触发，非语音）
- [ ] 模型路径配置 + 一次性生成（非流式）
- [ ] KSU 模块 sysconfig 电池白名单
- **验收**：能通过 UI 跟模型对话（非语音），llama-server 开机自启

### Phase 2：语音链路
- [ ] sherpa-onnx ASR 集成
- [ ] sherpa-onnx TTS 集成
- [ ] 前台服务 + AudioRecord
- [ ] 对话编排（手动按钮触发录音）
- **验收**：按按钮说话，能听到回答

### Phase 3：唤醒词
- [ ] openWakeWord 集成
- [ ] 持续监听 + 切换对话模式
- [ ] 回环防护
- **验收**：说"嘿小脑"自动启动对话

### Phase 4：稳定性
- [ ] Phantom Mic 配置文档
- [ ] KSU appops 自动配置
- [ ] 内存监控 + 自动卸载
- [ ] CI 自动构建
- **验收**：锁屏 24h 后仍可唤醒

### Phase 5：发布
- [ ] GitHub Release 自动化
- [ ] 完整文档
- [ ] 演示视频
- [ ] Issue 模板

### Phase 6：联网工具调用（设计草案 [09_phase6_network_tools_design.md](./09_phase6_network_tools_design.md)）
- [ ] 全局联网开关 + UI 设置
- [ ] Tool 接口 + ToolRouter + NetworkGate 基础设施
- [ ] 4 个联网工具：WeatherTool / TranslateTool / WebSearchTool / NewsTool
- [ ] 状态机扩展（新增 TOOL_RUNNING 状态）
- **验收**：开关打开后，说"明天天气"能联网查询并语音回答

### Phase 7：手机控制类（设计草案 [10_phase7_phone_control_design.md](./10_phase7_phone_control_design.md)）
- [ ] SystemSettingsTool（手电筒/亮度/静音等 10+ 设置）
- [ ] NotificationReaderTool（朗读 + 语音回复）
- [ ] AppLauncherTool L1（仅启动 app）
- [ ] ScreenCaptureTool（截屏 + vision API）
- **验收**：锁屏时来微信通知能朗读并语音回复

### Phase 8：平台化能力（设计草案 [11_phase8_platform_design.md](./11_phase8_platform_design.md)）
- [ ] Cap 1: 本地 API 暴露（端口配置 + 限流 + token 计数 + LAN 选项）
- [ ] Cap 3: 桌面小组件（一键对话 / 一键听写 / 状态卡）
- [ ] Cap 4: MCP 协议支持（暴露 8 个 MCP 工具给桌面端）
- **验收**：桌面端 Claude Desktop 能通过 MCP 调用手机上的 MiBrain 能力

### Phase 9：多模态（设计草案 [12_phase9_multimodal_design.md](./12_phase9_multimodal_design.md)）
- [ ] ImageChatTool（单轮 OCR + 多轮看图对话）
- [ ] 相机 intent + 图片 base64 处理
- [ ] 多轮上下文管理
- [ ] 状态机扩展（新增 IMAGE_CAPTURING 状态）
- **验收**：拍照后能多轮对话问图的内容

---

## 12. 知识产权与许可

- **本项目许可**：Apache 2.0
- **必须保留的 NOTICES**：
  - llama.cpp (MIT)
  - sherpa-onnx (Apache 2.0)
  - onnxruntime (MIT)
  - openWakeWord (MIT)
  - Compose / AndroidX (Apache 2.0)
- **不打包的第三方资产**：
  - 模型文件（用户自下，许可各自不同）
  - Phantom Mic APK（用户自装）

---

## 13. 开放问题（全部已确认）

> **更新（2026-06-30）**：原 6 个开放问题已全部在 [DECISIONS.md](../DECISIONS.md) 中确认，此处仅保留追溯。

| # | 原问题 | 决策 | 决策编号 |
|---|---|---|---|
| 1 | GitHub 仓库归属 | `qbjsdsb/mibrain` | [D12](../DECISIONS.md) |
| 2 | 项目名是否用 "MiBrain" | 是，确认 MiBrain | [D4](../DECISIONS.md) |
| 3 | 是否借鉴 ToolNeuron 代码结构 | 仅参考骨架，不 fork 代码（fork LlamaEngine.kt 路径已被 [X2](../DECISIONS.md) 废弃） | [X2](../DECISIONS.md) |
| 4 | 唤醒词定什么 | MVP 用英文 `hey_jarvis`，Phase 3 自训中文 | [D2](../DECISIONS.md) |
| 5 | 默认模型选哪个 | Qwen2.5-3B-Instruct Q4_K_M（备选 1.5B） | [D1](../DECISIONS.md) |
| 6 | 是否需要 RAG | MVP 不做，Phase 5 之后再说 | [D3](../DECISIONS.md) |

---

## 14. 当前阶段的明确边界

✅ **本阶段已交付**（Phase 0 设计冻结）：
- 完整设计文档 8 份（见 [docs/README.md](./README.md)）
- 决策清单 [DECISIONS.md](../DECISIONS.md)（15 条决策 + 5 条已废弃）
- 借鉴清单（本文 §3）+ 风险评估（本文 §10）+ 路线图（本文 §11）
- Issue 模板 3 份（bug / feature / test_report）
- 第三方许可声明 [third_party/NOTICES.md](../third_party/NOTICES.md)
- 三个目录骨架（[app/](../app/) / [ksu_module/](../ksu_module/) / [scripts/](../scripts/)）的 README 占位

❌ **本阶段未交付**（按用户要求"先不要交付"）：
- 任何 .kt / .sh / .gradle 代码
- 任何二进制（llama-server、.so、模型）
- 任何可执行物
- KSU 模块 zip
- APK
- GitHub Actions workflows（Phase 5 才补）

下一步等用户确认进入 Phase 1 后开始编码。
