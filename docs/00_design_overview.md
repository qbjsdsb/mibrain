# MiBrain - 红米 K50U 本地 AI 助手 项目设计文档

> 本文档**仅做设计，不交付代码**。先想清楚再动手。

---

## 0. 项目元信息

| 项 | 值 |
|---|---|
| 项目代号 | MiBrain（暂定） |
| 目标设备 | 红米 K50 Ultra（骁龙 8+ Gen 1，8GB RAM，Adreno 730，HyperOS） |
| Root 方案 | KernelSU + ZygiskNext + LSPosed |
| 用户硬约束 | 只能保证 KSU + ZygiskNext + LSP，其他能力不保证（不自编译内核、不改 ROM、不长期维护复杂 hook） |
| 核心目标 | 离线、本地、隐私不出手机的语音助手 |
| 开发语言 | Kotlin（APK）+ Shell（KSU 脚本）+ 现成 arm64 二进制 |
| 当前阶段 | 设计阶段，**不交付任何代码** |

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
| **Phantom Mic** | Mino260806/PhantomMic (LSPosed) | native hook AudioRecord.cpp，成熟 | 只 hook 不提供调度 |
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
| LLM 推理 | ToolNeuron 的 llama.cpp 集成 | JNI 包装、流式生成 | 参考 |
| TTS | ToolNeuron 的 sherpa-onnx 集成 | 10 声音、5 语言、onnx 模型加载 | 参考 |
| STT | ToolNeuron 的 sherpa-onnx ASR | 在线流式识别 + VAD | 参考 |
| 唤醒词 | Hotword Plugin 思路 | 持续监听 + 模型可换 | 重写为 Kotlin |
| LSPosed hook | WhatsMicFix 双 scope 架构 | app + system 同时 hook | 改造 |
| LSPosed native hook | Phantom Mic 的 AudioRecord.cpp hook | native 层注入 | 二次开发 |
| KSU 模块壳 | 标准模板 | service.sh / post-fs-data.sh / sepolicy.rule | 直接用 |
| Go 后端 | tasker-mcp 的 cross-compile | `GOOS=linux GOARCH=arm64` + CGO + NDK | 参考（如果选 Go） |

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
- ToolNeuron 已验证 llama.cpp + JNI 在 Android 上稳定
- GPU 加速：Adreno 730 支持 Vulkan，未来可启用 `-DLLAMA_VULKAN=ON`

**结果**：APK 内嵌 llama.cpp 预编译 .so（参考 ToolNeuron 的 jniLibs）

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
- 7.6MB APK，活跃维护（2024-07 v2.0）
- 我们不需要重写 hook，只要在 LSPosed 配置里启用即可

**结果**：不自己写 LSPosed 模块，**装现成的 Phantom Mic**，只在文档里给配置指引

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
│  │  │  ├─ llama.cpp JNI (本地 LLM 推理，流式输出)            │  │   │
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
│  │  ├─ service.sh：开机自启 + watchdog                          │   │
│  │  │   └─ am startservice MiBrainForegroundService              │   │
│  │  ├─ post-fs-data.sh：                                        │   │
│  │  │   ├─ 创建 /data/adb/mibrain/ 目录                         │   │
│  │  │   ├─ chcon SELinux 上下文                                 │   │
│  │  │   └─ appops set <uid> RECORD_AUDIO allow                 │   │
│  │  ├─ sepolicy.rule：放行 daemon 访问                          │   │
│  │  ├─ system/etc/sysconfig/mibrain.xml：电池白名单             │   │
│  │  └─ libs/：备用二进制（whisper.cpp / piper，可选）            │   │
│  └─────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────┘
                              ↑
                              │ 用户手动放入 /data/adb/mibrain/models/
                              │
┌────────────────────────────────────────────────────────────────────┐
│ 模型资产（不打包进模块，体积大）                                     │
│  ├─ qwen2.5-3b-instruct-q4_k_m.gguf   (~2GB，对话)                 │
│  ├─ sherpa-onnx-paraformer-zh (流式 ASR，~100MB)                  │
│  ├─ sherpa-onnx-vits-zh (TTS，~80MB)                               │
│  └─ openwakeword.onnx (唤醒词，~10MB)                              │
└────────────────────────────────────────────────────────────────────┘
```

---

## 6. 组件选型与版本

| 组件 | 选型 | 版本 | 来源 |
|---|---|---|---|
| LLM 推理 | llama.cpp | b9844+ | github.com/ggml-org/llama.cpp |
| Android JNI 集成 | llama.cpp android-arm64 包 | b9844 | 同上 release |
| 流式 ASR | sherpa-onnx | 1.10+ | github.com/k2-fsa/sherpa-onnx |
| 流式 TTS | sherpa-onnx VITS | 1.10+ | 同上 |
| 唤醒词 | openWakeWord via sherpa-onnx | 0.6+ | github.com/dscripka/openWakeWord |
| LSPosed hook | Phantom Mic | 2.0 | github.com/Mino260806/PhantomMic |
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
│   ├── 00_design_overview.md              # 本文档
│   ├── 01_architecture.md                 # 详细架构
│   ├── 02_build_guide.md                  # 编译指南
│   ├── 03_deploy_guide.md                 # 部署指南（用户向）
│   ├── 04_lspoded_setup.md                # LSPosed 配置指南
│   ├── 05_troubleshooting.md              # 故障排查
│   └── 06_performance_bench.md            # 性能基准
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
│   │   │   │   │   ├── LlamaEngine.kt              # llama.cpp JNI 封装
│   │   │   │   │   ├── SherpaAsrEngine.kt         # 流式 ASR
│   │   │   │   │   ├── SherpaTtsEngine.kt         # 流式 TTS
│   │   │   │   │   ├── WakeWordEngine.kt          # 唤醒词
│   │   │   │   │   └── ConversationEngine.kt      # 编排
│   │   │   │   ├── data/
│   │   │   │   │   ├── ModelManager.kt             # 模型管理
│   │   │   │   │   ├── SettingsRepository.kt
│   │   │   │   │   └── db/                          # Room
│   │   │   │   ├── ui/
│   │   │   │   │   ├── theme/
│   │   │   │   │   ├── screens/                    # 各 Compose 屏幕
│   │   │   │   │   └── components/
│   │   │   │   └── util/
│   │   │   ├── assets/
│   │   │   │   └── default_prompts.json            # 默认 system prompt
│   │   │   └── jniLibs/arm64-v8a/
│   │   │       ├── libllama.so                     # llama.cpp 预编译
│   │   │       ├── libsherpa-onnx-jni.so            # sherpa-onnx 预编译
│   │   │       └── libonnxruntime.so
│   │   └── proguard-rules.pro
│   ├── settings.gradle.kts
│   └── gradle/libs.versions.toml                    # 版本目录
│
├── ksu_module/                            # KSU 模块源码
│   ├── module.prop
│   ├── install.sh
│   ├── post-fs-data.sh
│   ├── service.sh
│   ├── uninstall.sh
│   ├── sepolicy.rule
│   ├── system/etc/sysconfig/mibrain.xml   # 电池白名单
│   └── build.sh                           # 打包 zip 脚本
│
├── scripts/                               # 辅助脚本
│   ├── download_models.sh                 # 一键下载模型
│   ├── download_binaries.sh               # 一键下载预编译 so
│   └── ci/build_apk.sh                    # CI 构建脚本
│
├── .github/workflows/                     # GitHub Actions
│   ├── build_apk.yml                      # 自动构建 APK
│   ├── build_ksu_module.yml               # 自动打包 KSU 模块
│   └── release.yml                        # 发布 release
│
└── third_party/                           # 第三方资源引用清单
    ├── NOTICES.md                         # 所有引用的开源项目
    └── licenses/
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
| HyperOS 升级 hook 失效 | 低 | 高 | 文档提示版本对应；多 hook 点备选 |
| llama.cpp 升级破坏 JNI | 中 | 中 | 锁版本到 b9844；CI 自动验证 |
| 麦克风被其他 App 占用 | 低 | 中 | 监听 AudioFocus 变化；占用时静默等待 |

---

## 11. 交付路线图

### Phase 0：设计冻结（当前）
- ✅ 本文档完成
- ⏳ 等待用户审阅
- ⏳ 等待用户给 GitHub 信息

### Phase 1：MVP 单链路打通（最小可用）
- [ ] Kotlin APK 骨架 + Compose UI（参考 ToolNeuron）
- [ ] llama.cpp JNI 集成（先用预编译 .so）
- [ ] 一个静态对话界面（按按钮触发，非语音）
- [ ] 模型加载 + 一次性生成
- [ ] KSU 模块壳（service.sh + sysconfig 白名单）
- **验收**：能通过 UI 跟模型对话（非语音）

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

## 13. 待用户确认的开放问题

在进入 Phase 1 之前需要用户回答：

1. **GitHub 仓库归属**：你的 GitHub 用户名是什么？沙箱里 gh 未配置，需要你提供 token 或自己 push
2. **项目名是否用 "MiBrain"**：还是有更好的命名？
3. **是否接受借鉴 ToolNeuron 的代码结构**：参考其骨架（不直接抄代码），仅借鉴架构思路
4. **唤醒词定什么**：默认"嘿小脑"，可改"小爱同学"会冲突，建议自定义
5. **默认模型选哪个**：Qwen2.5-3B（推荐）还是 Gemma3-1B（更省内存）？
6. **是否需要 RAG**：ToolNeuron 有，但增加复杂度。MVP 先不做？

---

## 14. 当前阶段的明确边界

✅ **本阶段已交付**：
- 本设计文档
- 完整架构图
- 借鉴清单
- 风险评估
- 路线图

❌ **本阶段未交付**（用户要求"先不要交付"）：
- 任何 .kt / .sh / .gradle 代码
- 任何二进制
- 任何可执行物
- KSU 模块 zip
- APK

下一步等用户审阅设计、回答第 13 节开放问题、提供 GitHub 信息后再进入 Phase 1。
