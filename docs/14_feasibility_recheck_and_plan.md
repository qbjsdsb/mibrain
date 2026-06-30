# MiBrain 可行性复审 + 分阶段实施计划

> 本文档在 Phase 0 设计冻结后，对全部关键依赖做最终可行性复审，并基于复审结果给出严谨的分阶段实施计划。
>
> 复审方式：3 个并行 subagent 实测 GitHub / HuggingFace / 官方文档站，每个验证项明确标 ✅ 已验证 / ⚠️ 部分验证有疑问 / ❌ 与文档不符。
>
> 复审日期：2026-06-30

---

## 一、复审汇总

### 1.1 已确认可行 ✅（无需调整）

| 项 | 验证结论 | 来源 |
|---|---|---|
| llama.cpp b9844 | 真实 release tag（今日刚发布），MIT 许可 | [release b9844](https://github.com/ggml-org/llama.cpp/releases/tag/b9844) |
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

### 1.2 必须修订的严重错误 ❌

| # | 错误 | 真实情况 | 影响范围 |
|---|---|---|---|
| E1 | KWS 模型名 `sherpa-onnx-keyword-spotting-zh-vgg` | **不存在**！真实为 `sherpa-onnx-kws-zipformer-wenetspeech-3.3M-2024-01-01`（中文）或 `sherpa-onnx-kws-zipformer-zh-en-3M-2025-12-20`（中英双语，更新更小） | 全项目所有提到 KWS 模型的文档 |
| E2 | ToolNeuron `LlamaEngine.kt` | **不存在**！真实文件为 `InferenceService.kt` (57.8KB) + `InferenceClient.kt` (43.9KB)，位于 `app/src/main/java/com/dark/tool_neuron/service/inference/` | 全项目所有引用 LlamaEngine.kt 的文档 |
| E3 | llama.android 路径 `tree/master/llama.android` | **404**！正确路径 `tree/master/examples/llama.android/` | README / NOTICES / 00 / 01 / 02 / DECISIONS |
| E4 | vits-zh-ll 声称 Apache 2.0 | HF 卡 metadata 缺失，**官方未声明许可**，训练数据来源不明 | D22 / 00 / 01 / 05 / 07 / NOTICES |
| E5 | Android opencl-adreno 包 | **仅 Windows 版**（`-bin-win-opencl-adreno-arm64.zip`）；Android Adreno 加速需自行用 Snapdragon 工具链编译 | 01_feasibility_verification.md §2.1 |
| E6 | streaming-zipformer-bilingual-zh-en 命名 | 真实文件名带日期后缀 `-2023-02-20` | 多份文档 |

### 1.3 估算偏差 ⚠️

| # | 文档原值 | 真实值 | 影响评估 |
|---|---|---|---|
| W1 | libllama.so + libggml.so ~50MB | **~30MB**（日本 Pasona 实测） | D7 代价估算偏高，APK 体积比预期小 |
| W2 | ToolNeuron 258★ | **429★**（GitHub API 实时） | 借鉴价值被低估 |
| W3 | ToolNeuron Apache 2.0 | **MIT**（2026-04-22 从 GPLv3 切到 MIT） | NOTICES 已正确，但其他文档可能有误 |

### 1.4 风险升级与新发现

| # | 风险 | 评级 | 缓解 |
|---|---|---|---|
| R1 | **TTS 无 chunk 回调**：sherpa-onnx TTS 是 OfflineTts（一次性生成完整 PCM），无官方流式回调 | 中 | Phase 2 / Phase 9 G6 字幕同步需自切 PCM 数组成 chunk 喂 AudioTrack |
| R2 | **Qwen2.5-1.5B function calling brittle**：BFCL v3 实测直答准确率 44%，需 32-token CoT 才到 64%，长 CoT 反崩到 25% | 高 | Phase 6 工具调用需 32-token CoT prompt 或换 3B；M1.5B 复杂工具调用不可靠 |
| R3 | **Direct Boot + sherpa-onnx 组合未公开验证**：理论可行但无先例，4 个子项需 PoC | 高 | Phase 1 启动前必做 PoC |
| R4 | **Phantom Mic 高风险评级准确甚至偏保守**：最后 release 2024-07-24，近 2 年未更新，无 HyperOS 2 / Android 15 兼容性反馈，**无对等替代品** | 高 | D14 已准确，无新缓解方案；Phase 4 真机验证为硬关卡 |
| R5 | **Qwen 商标限制**：Apache 2.0 允许商用但 "Qwen" 商标不可用于衍生品市场宣传 | 低 | README / UI 不写 "Powered by Qwen"，改写 "使用 Qwen2.5 模型" |
| R6 | **sherpa-onnx v1.13.3 PyPI wheel 未上传**：仅 GitHub 源码 + 二进制资产，PyPI 仍是 1.13.2 | 低 | Android APK 直接用 AAR，不依赖 PyPI |

---

## 二、分阶段实施计划

> 基于可行性复审结果，将原 Phase 1-10 重组为 12 个 Stage，每个 Stage 设明确**准入条件** / **交付物** / **准出条件** / **风险关卡**。
>
> 关键原则：
> 1. **未通过可行性补救（Stage 0）不启动 Phase 1**
> 2. **未通过 PoC（Stage 1）不写正式代码**
> 3. **未通过真机验证（Phase 4）不发布**
> 4. **每 Stage 设回退路径**，避免卡死

### Stage 0：可行性补救（必须先做，预计 1-2 天）

**目标**：修复 1.2 节 6 项严重错误 + 1.3 节 3 项估算偏差，让所有文档回到可信状态。

**交付物**：
- [ ] 全项目搜索 `keyword-spotting-zh-vgg` → 替换为 `kws-zipformer-wenetspeech-3.3M-2024-01-01`（中文默认）或 `kws-zipformer-zh-en-3M-2025-12-20`（中英双语）
- [ ] 全项目搜索 `LlamaEngine.kt`（在 ToolNeuron 引用语境下）→ 改为 `InferenceService.kt + InferenceClient.kt`
- [ ] 全项目搜索 `tree/master/llama.android` → 改为 `tree/master/examples/llama.android`
- [ ] 修订 D22：vits-zh-ll 改为"社区贡献，许可未明确声明"，**不可声称 Apache 2.0**；备选改为 `matcha-icefall-zh-baker`（已验证 Apache 2.0）
- [ ] 修订 01_feasibility_verification.md §2.1：删除 "Android opencl-adreno-arm64" 表述，改为 "Adreno 加速需自行用 Snapdragon 工具链编译"
- [ ] 修订 streaming-zipformer 文件名：补 `-2023-02-20` 后缀
- [ ] 修订 libllama.so 体积：~50MB → ~30MB
- [ ] 修订 ToolNeuron star 数：258 → 429
- [ ] 新增风险表条目：TTS chunk / Qwen 1.5B function calling / Direct Boot 未验证
- [ ] 在 README.md 加 "Qwen 商标不可用于衍生品宣传" 提示

**准出条件**：所有文档无幻觉引用，所有依赖链接可达，所有许可声明准确。

**回退路径**：无（这是补救，不通过则整个项目可信度崩塌）。

### Stage 1：关键 PoC（Phase 1 启动前必做，预计 3-5 天）

**目标**：用最小代码验证三个高风险技术点，避免 Phase 1 写了一半发现根因性问题。

**3 个 PoC 并行**：

#### PoC-A：Direct Boot + sherpa-onnx 加载链路
- 写一个最小 APK：Application directBootAware + ForegroundService directBootAware
- 在锁屏前重启手机，验证：
  1. `ACTION_LOCKED_BOOT_COMPLETED` 广播能收到
  2. `Context.createDeviceProtectedStorageContext().openFileInput()` 能读 DE 区文件
  3. `System.loadLibrary("sherpa-onnx-jni")` 在 Direct Boot 下能加载 .so
  4. sherpa-onnx OfflineRecognizer 能从 DE 路径加载模型并完成一次推理
  5. HyperOS 2 的"超级省电"不在解锁前杀 FGS
- **不通过** → 退路：放弃 Direct Boot 模式，回退到 "用户首次解锁后才启动"（牺牲锁屏唤醒能力，但保留其他所有功能）

#### PoC-B：Phantom Mic 在 HyperOS 2 上的兼容性
- 装 Phantom Mic v2.0
- LSPosed 启用 + 作用域勾选测试 APK
- 锁屏后 AudioRecord.read() 是否返回真实数据
- **不通过** → 退路：
  1. 评估替代品（搜索上游 Issues / 其他 hook 项目）
  2. 退到 appops + 双触发兜底（06_lspoded_setup.md §6.1 方案 B）
  3. 最坏情况：放弃锁屏唤醒，仅亮屏可用

#### PoC-C：JNI wrapper 编译可行性
- 拉 llama.cpp b9844 源码
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

**回退路径**：3B 模型 OOM 时切回 1.5B；JNI 编译失败用 tarball .so。

### Stage 3：Phase 2 语音链路（预计 2-3 周）

**准入**：Stage 2 准出。

**目标**：完整对话循环 IDLE→LISTENING→THINKING→SPEAKING→COOLDOWN→IDLE。

**交付物**：
- [ ] sherpa-onnx StreamingAsr 集成（zipformer-bilingual-zh-en-2023-02-20）
- [ ] sherpa-onnx OfflineTts 集成（vits-zh-ll，**注意无 chunk 回调**，需自切 PCM 喂 AudioTrack）
- [ ] sherpa-onnx VoiceActivityDetector 集成（silero_vad）
- [ ] 前台服务 + WakeLock + 状态机 CAS
- [ ] 录音权限请求 UI

**关键风险**：
- TTS 无 chunk 回调 → MVP 用"先合成完整 PCM 再播"模式（延迟略大但简单）
- 若需"边合成边播"，Kotlin 端按 200ms 切 PCM 数组成 chunk

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
- [ ] KWS 模型选 `kws-zipformer-wenetspeech-3.3M-2024-01-01`（中文默认）
- [ ] text2token 生成 "嘿小脑" / "你好小脑" keywords.txt
- [ ] 调参：boosting_score 1.5-3.5 + threshold 0.25-0.6
- [ ] KWS 与 ASR 切换不冲突（IDLE 用 KWS，LISTENING 用 ASR）
- [ ] 中文唤醒词样本采集（[D15](../DECISIONS.md)）

**准出条件**：
- 唤醒率 > 80%（10 次说 8 次以上触发）
- 误唤醒 < 1 次/小时
- KWS 占用 CPU < 5%

**回退路径**：自训中文效果差 → MVP 用英文 `hey_jarvis`（[D2](../DECISIONS.md)）+ 桌面快捷方式兜底。

### Stage 5：Phase 4 稳定性（预计 2-3 周，硬关卡）

**准入**：Stage 4 准出 + Stage 1 PoC-B 通过。

**目标**：锁屏唤醒可用 + 内存可控 + 不被杀。

**交付物**：
- [ ] Phantom Mic 真机验证（依赖 PoC-B 通过）
- [ ] appops 自动授权（KSU post-fs-data.sh）
- [ ] 8 步 MIUI 白名单自动化检测
- [ ] 内存监控（lmkd 阈值告警）
- [ ] 真机连续 24 小时稳定性测试
- [ ] 发热监控 + 降频策略
- [ ] 诊断脚本 [07_troubleshooting.md §8.1](./07_troubleshooting.md)

**准出条件**（硬关卡）：
- 锁屏 1 小时后唤醒率 > 70%
- 24 小时无崩溃重启
- 峰值内存 < 7GB（[D1](../DECISIONS.md) 1.5B 实测 6.77GB）
- 8+ Gen 1 满载温度 < 75℃

**回退路径**：Phantom Mic 不通过 → 仅亮屏可用 + appops 兜底；温度过高 → 限制每日对话时长。

### Stage 6：Phase 5 发布（预计 1-2 周）

**准入**：Stage 5 硬关卡通过。

**交付物**：
- [ ] CI/CD（GitHub Actions：APK 自动构建 + KSU zip 打包）
- [ ] Release v0.1.0：APK + KSU zip + Phantom Mic 安装指引
- [ ] 性能基准 [08_performance_bench.md](./08_performance_bench.md)
- [ ] 用户文档（[05_deploy_guide.md](./05_deploy_guide.md) 已就绪）
- [ ] Issue 模板 + 用户反馈通道

**准出条件**：
- 至少 1 名外部用户（非开发者）按文档独立部署成功
- 性能基准数据完整（tok/s / RTF / 内存 / 温度 / 唤醒率）

**回退路径**：外部用户反馈严重问题 → 回到 Stage 5 修复。

### Stage 7-10：Phase 6-9 扩展功能（每 Phase 预计 2-4 周）

| Stage | Phase | 准入 | 核心交付 | 关键风险 |
|---|---|---|---|---|
| 7 | Phase 6 联网工具 | Stage 6 | 全局开关 + 4 工具 + **Qwen 1.5B function calling 用 32-token CoT prompt**（[R2](#14-风险升级与新发现)） | 1.5B FC 准确率 44%→64%，复杂工具不可靠 |
| 8 | Phase 7 手机控制 | Stage 7 | 4 工具（通知朗读 / 应用启动 monkey / 系统设置 / 截屏看屏） | A2A 协议推进中，微信回复路径需关注 |
| 9 | Phase 8 平台化 | Stage 8 | NanoHTTPD 内嵌 + Widget + MCP server | - |
| 10 | Phase 9 多模态 | Stage 9 | ImageChatTool + IMAGE_CAPTURING 状态机 + GLM-4V | 多模态 API 需联网 + 联网开关联动 |

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
| Gate 3 | Stage 7 Phase 6 工具调用 | 1.5B FC 不够 → 切 3B 或加 CoT prompt |

---

## 四、决策修订清单（Stage 0 必须先做）

基于复审结果，需对 [DECISIONS.md](../DECISIONS.md) 做以下修订：

| 决策 | 修订内容 |
|---|---|
| D7 | 修正 llama.android 路径为 `examples/llama.android`；ToolNeuron 参考文件改为 `InferenceService.kt` + `InferenceClient.kt`（非 LlamaEngine.kt）；libllama.so 体积 ~50MB→~30MB |
| D22 | vits-zh-ll 改为"社区贡献，许可未明确声明"；备选升级为 `matcha-icefall-zh-baker`（Apache 2.0 已验证）；ASR 文件名补 `-2023-02-20` 后缀 |
| D23 | KWS 模型从虚构的 `zh-vgg` 改为 `kws-zipformer-wenetspeech-3.3M-2024-01-01`（中文默认）；补记 text2token 自定义词流程 |
| 新增 D24 | Qwen 商标限制：不写 "Powered by Qwen"，改 "使用 Qwen2.5 模型" |
| 新增 D25 | Qwen2.5-1.5B function calling 限制：Phase 6 工具调用需 32-token CoT prompt 或换 3B |
| 新增 D26 | TTS 无 chunk 回调：MVP 用 "先合成完整 PCM 再播"，Phase 10 G6 字幕按 200ms 切 PCM |
| 新增 D27 | Direct Boot + sherpa-onnx 组合未公开验证，Stage 1 PoC-A 为硬关卡 |

---

## 五、立即行动项

按优先级排序，需立即执行：

1. **Stage 0 全部修订**（见上方 9 项交付物）— 优先级 P0
2. **Stage 1 PoC 计划文档化**（PoC-A/B/C 的具体测试用例）— 优先级 P1
3. **DECISIONS.md 新增 D24-D27** — 优先级 P0
4. **各文档交叉修订**（搜索幻觉引用） — 优先级 P0

---

## 六、关键来源

- [llama.cpp b9844 release](https://github.com/ggml-org/llama.cpp/releases/tag/b9844)
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
