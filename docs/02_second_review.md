# 第二轮严谨审视：5 个新发现与修订

> 审视日期：2026-06-30
> 触发条件：用户要求"反复检查设计是否可用，借鉴的项目都不要忘"
> 结论：发现 5 个之前漏掉的问题，全部修订并并入设计

---

## 发现 1：fork ToolNeuron 的 LlamaEngine.kt 路径不成立

### 之前的判断
"直接 fork ToolNeuron 的 LlamaEngine.kt（MIT 许可），省 1500+ 行 Kotlin 代码量"

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

---

## 发现 4：Phantom Mic 在 Android 14+ 的兼容性未验证

### 之前的判断
"Phantom Mic v2.0 已经验证可达"

### 重新查证
- 红米 K50U 原厂 HyperOS 1 基于 Android 12
- 现役大概率已升级到 **HyperOS 2 基于 Android 15**
- Android 14+ 引入了 `foregroundServiceType="microphone"` 强制要求
- Phantom Mic 项目 README 未明确声明 Android 14/15 兼容

### 风险等级
🟡 中（可能能用，需要真机验证）

### 修订
1. Phase 1-3 不依赖 Phantom Mic，先做"按按钮触发"的 APK
2. Phase 4 真机验证 Phantom Mic 是否需要更新版本或备选模块
3. 增加备选方案：`appops set <uid> RECORD_AUDIO allow`（root shell 命令）

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
- 已验证 `sherpa-onnx-keyword-spotting-zh-vgg.tar.bz2` 可下载
- 但默认关键词是预设的，自定义仍需训练

---

## 汇总：对原设计的修订

| 原设计 | 修订为 | 修订理由 |
|---|---|---|
| APK 内嵌 llama.cpp JNI | 改用 HTTP 调用 llama-server 二进制 | 发现 1：JNI 工程量太大 |
| 模型放 `/data/adb/mibrain/models/` | 改为运行时下载到 app 私有目录 | 发现 2：跨 SELinux 域访问麻烦 |
| sherpa-onnx Kotlin API 待确认 | 确认是官方维护，直接用 | 发现 3：调研完成 |
| Phantom Mic 解决一切录音问题 | 改为 Phase 4 才验证 | 发现 4：兼容性未验证 |
| 唤醒词用"嘿小脑" | MVP 用 hey_jarvis 或手动触发 | 发现 5：中文需自训模型 |

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

---

## 结论

第二轮审视发现 5 个真实问题，全部已修订并并入设计。**整体方案可行性不下降，工程量反而降低**（跳过 JNI 集成）。

可以进入 Phase 1。
