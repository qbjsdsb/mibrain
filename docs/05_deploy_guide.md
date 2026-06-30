# 部署指南（用户向）

> 本文档面向最终用户：拿到 MiBrain 后如何在红米 K50 Ultra 上跑起来。

> **修订说明（2026-06-30）**：本文档已按最新决策更新，主要变更：
> - **D7 修订**：推理后端从 llama-server HTTP 切回 JNI（参考 llama.android 官方模块 + ToolNeuron `InferenceService.kt` + `InferenceClient.kt`（位于 `service/inference/` 目录）），不再有独立子进程
> - **X7 废弃**：llama-server HTTP 路径废弃，所有 `/health` 检查、`llama-server.log` 引用、`llama.cpp 二进制升级` 章节均失效
> - **D21 新增**：模型存储路径改为 `/data/user_de/0/com.mibrain/files/models/`（DE 加密区 + Direct Boot），用户不再需要手动 mkdir `/sdcard/MiBrain/models/`
> - **D22 新增**：ASR/TTS 模型换 Apache 2.0 许可的 sherpa-onnx 官方模型（弃用 paraformer/aishell3）
> - **D23 新增**：唤醒词改 sherpa-onnx KWS（弃用 openWakeWord）
> - **D1 修订**：默认模型从 3B 改为 1.5B Q4_K_M（~1GB），3B 作为质量优先可选
>
> 详见 [DECISIONS.md](../DECISIONS.md)。

---

## 0. 前置条件检查

在开始之前，确认你的设备满足：

| 项 | 要求 | 验证方式 |
|---|---|---|
| 设备 | 红米 K50 Ultra（骁龙 8+ Gen 1） | 设置 → 我的设备 |
| 系统 | **HyperOS 3**（Android 15，[D6](../DECISIONS.md)）| 设置 → 我的设备 → MIUI 版本（应为 OS3.0.1.0.VLFCNXM 或更高；2026-01-20 已对 K50U 推送） |
| Root | KernelSU 已装且工作 | KSU Manager 应用可打开 |
| Zygisk | ZygiskNext 已装且启用 | LSPosed Vector 显示"Zygisk 已注入" |
| LSPosed | **LSPosed Vector v2.0.3-7716**（[D9](../DECISIONS.md)，2026-05-20 最新版）| LSPosed Vector Manager 显示框架激活；支持 Android 8.1-17 Beta 3 |
| 存储 | 至少 5GB 可用空间 | 文件管理器查看 |
| 网络 | 首次配置需联网下载 ~1.4GB 模型（默认配置） | - |

> ⚠️ **2026-06-30 第三轮 web 核实修订**（[F2](./14_feasibility_recheck_and_plan.md)）：
> - 系统：原"HyperOS 1+"升级为"HyperOS 3"。HyperOS 3 国行版已对 K50U 推送：OS3.0.1.0.VLFCNXM（2026-01-20 OTA）/ OS3.0.2.0.VLFCNXM（2026-04-13 Fastboot 包）。如未升级，先在系统更新里升级，再做后续步骤。
> - LSPosed：原"LSPosed"升级为"LSPosed Vector v2.0.3-7716"。LSPosed 原项目自 2024 年起活跃度下降，社区接力维护分支 **LSPosed Vector** 是当前唯一活跃维护版本（[D9](../DECISIONS.md)）。如已装 LSPosed 原版，先卸载再装 LSPosed Vector。

不满足任何一项 → 先解决再继续。

---

## 1. 下载所需文件

### 1.1 MiBrain 仓库代码

```bash
# 在电脑上
git clone https://github.com/qbjsdsb/mibrain.git
```

### 1.2 二进制依赖

最终用户不需要单独下载 llama-server 或 sherpa-onnx AAR：

- **llama-server**：JNI 路径下不再需要单独二进制，`libllama.so` + `libggml.so` 已打入 APK 的 `jniLibs/arm64-v8a/`（[D7](../DECISIONS.md)、[X7](../DECISIONS.md)）
- **sherpa-onnx AAR**：由开发者构建 APK 时作为依赖打入，用户不直接下（[D8](../DECISIONS.md)）

最终用户只需准备三样东西：

| 文件 | 来源 | 大小 | 用途 |
|---|---|---|---|
| MiBrain APK | 项目 Release 页或自行构建（见 §4.1） | ~80MB | 主应用（含 libllama.so + libggml.so + sherpa-onnx .so） |
| KSU 模块 zip | 项目 Release 页或自行构建（见 §3.1） | <1MB | root 权限与 appops 配置 |
| Phantom Mic APK | [Xposed-Modules-Repo](https://github.com/Xposed-Modules-Repo/tn.amin.phantom_mic/releases) | ~5MB | 录音 hook（见 [06_lspoded_setup.md](./06_lspoded_setup.md)） |

### 1.3 模型文件

> **重要**：默认情况下用户不需要手动下载模型。APK 首次启动会引导下载到 DE 加密区（[D21](../DECISIONS.md)）。下表仅供了解组成与可选 adb 预下流程使用。

| 文件 | 下载地址 | 大小 | 用途 |
|---|---|---|---|
| Qwen2.5-1.5B-Instruct Q4_K_M（默认） | https://huggingface.co/Qwen/Qwen2.5-1.5B-Instruct-GGUF/resolve/main/qwen2.5-1.5b-instruct-q4_k_m.gguf | ~1GB | 默认对话模型（[D1](../DECISIONS.md)） |
| Qwen2.5-3B-Instruct Q4_K_M（备选） | https://huggingface.co/Qwen/Qwen2.5-3B-Instruct-GGUF/resolve/main/qwen2.5-3b-instruct-q4_k_m.gguf | ~2GB | 质量优先可选 |
| sherpa-onnx streaming-zipformer-bilingual-zh-en ASR | github.com/k2-fsa/sherpa-onnx/releases/download/asr-models/sherpa-onnx-streaming-zipformer-bilingual-zh-en.tar.bz2 | ~250MB | 语音识别（Apache 2.0，[D22](../DECISIONS.md)） |
| sherpa-onnx vits-zh-ll 中文 TTS | huggingface.co/k2-fsa/sherpa-onnx/resolve/main/tts-models/sherpa-onnx-vits-zh-ll.tar.bz2 | ~150MB | 中文 TTS（社区贡献，许可未明确声明；HF 卡 metadata 缺失，[D22](../DECISIONS.md)） |
| silero_vad.onnx | sherpa-onnx release 自带 | ~10MB | VAD |
| sherpa-onnx KWS zipformer-wenetspeech | github.com/k2-fsa/sherpa-onnx/releases/download/kws-models/sherpa-onnx-kws-zipformer-wenetspeech-3.3M-2024-01-01.tar.bz2 | ~10MB | 唤醒词 KWS（[D23](../DECISIONS.md)；原引用 `zh-vgg` 不存在，真实模型为 zipformer 架构） |

合计默认配置约 1.4GB。

### 1.4 SHA256 校验

下载完后**必须校验 SHA256**，避免下错版本或损坏：

```bash
# 期望值（Phase 1 release 时填入，目前是占位）
sha256sum qwen2.5-1.5b-instruct-q4_k_m.gguf
# 期望: <Phase 1 release 时公布>
```

校验失败的文件不要用，重新下载。

---

## 2. 准备手机存储结构

> **修订（[D21](../DECISIONS.md)）**：模型路径已改为 APK 内部管理，落在 DE 加密区
> `/data/user_de/0/com.mibrain/files/models/`，用户**不再需要**手动 `mkdir /sdcard/MiBrain/models/`。
>
> 选 DE 区的原因：
> 1. FBE 加密下，CE 区（如 `/sdcard/`）在用户首次解锁前不可读，开机自启会失败
> 2. DE 区在开机即可读，配合 Direct Boot 让锁屏唤醒在加密设备上也能达成
> 3. DE 区属于 `app_data_file` SELinux 类型，JNI 调用不跨域（[D7](../DECISIONS.md)）

**默认流程（推荐）**：APK 首次启动 → 检测 DE 区模型 → 不存在则触发内置下载器（带进度 UI + 断点续传 + SHA256 校验），下载到上述路径。**用户无需手动操作**。

**可选流程（adb push 提前下好模型）**：网络较慢或希望首次启动更快可走此流程。模型解压后整体 push 到 DE 区：

```bash
# 模型解压后预期结构（具体以 sherpa-onnx 官方包内结构为准）
# models/
# ├── qwen2.5-1.5b-instruct-q4_k_m.gguf
# ├── silero_vad.onnx
# ├── streaming-zipformer-bilingual-zh-en/
# ├── vits-zh-ll/
# └── kws-zipformer-wenetspeech-3.3M-2024-01-01/

# 先 push 到 /data/local/tmp/（中转，所有用户可写）
adb push models /data/local/tmp/mibrain_models

# 进入 shell 用 root 拷到 DE 区
adb shell
su
PKG=com.mibrain
DEST=/data/user_de/0/$PKG/files/models
mkdir -p $DEST
cp -r /data/local/tmp/mibrain_models/. $DEST/
chown -R $(stat -c %u /data/user_de/0/$PKG):$(stat -c %g /data/user_de/0/$PKG) $DEST
chmod -R 0700 $DEST
ls -la $DEST
```

注意：DE 区只能由对应 APK 的 UID 或 root 访问，**不要**直接 `adb push` 到 DE 路径（adb shell 的 shell 用户无权写）。

APK 首次启动检测到模型已就位会跳过下载，直接进入加载流程。

---

## 3. 安装 KSU 模块

### 3.1 构建 KSU 模块 zip

```bash
cd mibrain
# Phase 1 完成后会有
./scripts/build_ksu_zip.sh
# 产出: ksu_module/mibrain-<version>.zip
```

### 3.2 在 KSU Manager 安装

1. 打开 KernelSU Manager
2. 模块 → 从存储安装
3. 选择 `mibrain-<version>.zip`
4. 重启手机

### 3.3 验证 KSU 模块工作

```bash
adb shell
# 检查模块目录
ls /data/adb/modules/mibrain/
# 期望: module.prop, post-fs-data.sh, sepolicy.rule, ...

# 检查 appops 是否生效（post-fs-data.sh 应该已执行）
appops get <mibrain_uid> RECORD_AUDIO
# 期望: RECORD_AUDIO: allow; time=...

# 检查日志
cat /data/adb/mibrain/install.log | tail -20
```

> **修订（[D7](../DECISIONS.md)、[X7](../DECISIONS.md)）**：JNI 路径下不再有独立子进程，因此**没有** `/data/adb/mibrain/llama-server.log`，也**没有** `curl /health` 检查。LLM 推理在 APK 进程内通过 JNI 完成。

如果模块目录缺失或 appops 未生效，看 [07_troubleshooting.md](./07_troubleshooting.md)。

---

## 4. 安装 MiBrain APK

### 4.1 构建 APK

```bash
cd mibrain/app
./gradlew assembleRelease
# 产出: app/build/outputs/apk/release/app-release.apk
```

### 4.2 安装到手机

```bash
adb install -r app-release.apk
```

### 4.3 首次启动配置

1. 打开 MiBrain 应用
2. 系统会请求以下权限，**全部允许**：
   - 录音权限
   - 通知权限（Android 13+）
   - 后台自启动（手动跳转）
3. 进入"模型设置"，确认路径自动检测到 DE 区（`/data/user_de/0/com.mibrain/files/models/`）
4. 点"下载模型"，等待 APK 内置下载器完成（首次约 1.4GB，建议 Wi-Fi；如已按 §2 可选流程 adb push，则跳过下载）
5. SHA256 校验通过后，点"加载模型"，应该显示 **LlamaEngine 加载成功**
6. 启用前台服务

> **关于 Direct Boot（[D21](../DECISIONS.md)）**：APK 应在 `AndroidManifest.xml` 声明
> `android:directBootAware="true"`，对应的前台服务 / Receiver 也加 `directBootAware`。
> 这样锁屏前（用户首次解锁前）也能启动，保证开机自启与锁屏唤醒链路在 FBE 加密设备上可达。
> 具体声明以 Phase 1 实现为准。

---

## 5. 关键：MIUI/HyperOS 系统设置白名单（必做）

**这一步漏一个都会导致锁屏后无法唤醒**。在 HyperOS 上做完以下 **9 步**（[E13](./14_feasibility_recheck_and_plan.md) 修订，HyperOS 2.0+ 新增"应用智能休眠"开关必须关闭）：

### 5.1 自启动
- 路径：设置 → 应用 → 应用管理 → MiBrain → 自启动
- 操作：**打开**

### 5.2 省电策略
- 路径：设置 → 应用 → 应用管理 → MiBrain → 省电策略
- 操作：**无限制**

### 5.3 应用电池使用
- 路径：电池 → 应用电池使用 → MiBrain
- 操作：**无限制**

### 5.4 隐私保护 → 电池优化
- 路径：设置 → 隐私保护 → 特殊权限 → 电池优化
- 操作：找到 MiBrain → **不优化**

### 5.5 神隐模式（HyperOS 1 才有）
- 路径：设置 → 电池 → 神隐模式
- 操作：MiBrain 设为"无限制"

### 5.6 最近任务锁定
- 操作：打开 MiBrain → 进最近任务 → 长按 MiBrain 卡片 → 锁定（出现锁图标）

### 5.7 通知权限
- 路径：设置 → 通知 → MiBrain
- 操作：**允许通知**，前台服务必须

### 5.8 录音权限（root 强制）
KSU 模块 `post-fs-data.sh` 会自动执行：
```bash
appops set <mibrain_uid> RECORD_AUDIO allow
```
但也可以手动：
```bash
adb shell
su
pm list packages -U com.mibrain | awk '{print $2}' | cut -d/ -f2
# 输出类似 uid=10234
appops set 10234 RECORD_AUDIO allow
appops get 10234 RECORD_AUDIO
# 应该输出: RECORD_AUDIO: allow; time=...
```

> **关于 appops 与 Phantom Mic 的关系**（[E13](./14_feasibility_recheck_and_plan.md) 修订）：appops 是 Phantom Mic 失效时的 fallback，**不是省略首次授权的捷径**。即使 appops 已 allow，首次安装 MiBrain 仍需在系统设置里手动授予"录音权限"运行时授权（Android 13+ 流程），appops 仅在后续被 MIUI 收回时自动恢复。
> Phase 4 真机验证前，**先做 [D14 复查](../DECISIONS.md)**：去 Phantom Mic 上游仓库
> 看是否有新版本 / HyperOS 3（Android 15）兼容性反馈，若上游已停滞需评估换备选方案
> （[06_lspoded_setup.md §6.1](./06_lspoded_setup.md) 列了候选）。在复查未完成前不要把
> appops 当成可省略步骤——锁屏唤醒链路里它仍是兜底。

### 5.9 应用智能休眠（HyperOS 2.0+ 必做，[E13](./14_feasibility_recheck_and_plan.md) 新增）

> ⚠️ **2026-06-30 第三轮新增**：HyperOS 2.0+ 引入 AI 智能休眠机制，**默认会自动休眠后台长期不活跃的应用**，即使已加入电池白名单也会被杀。MiBrain 作为后台长期运行的语音助手，**必须关闭此项**。

- 路径：设置 → 电池 → 应用智能休眠（或 设置 → 应用 → 应用管理 → MiBrain → 应用智能休眠）
- 操作：**关闭**（或把 MiBrain 加入"不智能休眠"白名单）
- 验证：锁屏 1 小时后 adb logcat 仍能看到 MiBrain ForegroundService 在跑

> 关闭此项与 §5.2/§5.3 "无限制" 不可互替——三者各自独立的开关：
> - §5.2/§5.3 "无限制"：传统省电策略，Android 6+ 就有
> - §5.9 "应用智能休眠"：HyperOS 2.0+ 新增的 AI 行为预测休眠，**白名单不豁免**
>
> 不关 §5.9 的现象：锁屏 30 分钟-1 小时后 ForegroundService 被杀，唤醒失败，需重启用。

### 验证白名单生效
锁屏 5 分钟后，adb logcat 应该还能看到 MiBrain ForegroundService 在跑：
```bash
adb logcat | grep MiBrain
```

---

## 6. 安装 LSPosed Vector Phantom Mic 模块

详见 [06_lspoded_setup.md](./06_lspoded_setup.md)。

简版步骤：
1. 下载 https://github.com/Xposed-Modules-Repo/tn.amin.phantom_mic/releases/download/3-2.0/PhantomMic-2.0.apk
2. `adb install PhantomMic-2.0.apk`
3. 打开 LSPosed Vector Manager → 模块 → 启用 Phantom Mic
4. 在 Phantom Mic 范围里勾选 MiBrain
5. 重启手机

---

## 7. 验证完整链路

### 7.1 测试对话
1. 打开 MiBrain
2. 点"开始监听"
3. 说唤醒词（具体词以 sherpa-onnx KWS 模型示例词为准，[D23](../DECISIONS.md)）
4. 听到回应"嗯？"
5. 说"今天天气"
6. 听到 AI 回答

### 7.2 测试锁屏唤醒
1. 锁屏
2. 等 10 秒
3. 说唤醒词
4. 应该能触发

如果锁屏后唤不醒 → [07_troubleshooting.md](./07_troubleshooting.md)。

### 7.3 测试内存压力
```bash
adb shell
free -m
# 唤醒对话时观察可用内存
```

应该 >300MB 可用，不会触发 lmkd。

---

## 8. 卸载

### 8.1 卸载 APK
```bash
adb uninstall com.mibrain
```

### 8.2 卸载 KSU 模块
KSU Manager → 模块 → MiBrain → 删除 → 重启

### 8.3 卸载 Phantom Mic
LSPosed Vector → 禁用 → 卸载 APK

### 8.4 清理模型（可选）
```bash
adb shell
su
rm -rf /data/user_de/0/com.mibrain/files/models
```

---

## 9. 升级

### 9.1 升级 MiBrain APK
直接 `adb install -r` 覆盖安装，配置保留。

### 9.2 升级 KSU 模块
KSU Manager → 模块 → 从存储安装新版 zip → 重启

### 9.3 升级推理后端
> **修订（[D7](../DECISIONS.md)、[X7](../DECISIONS.md)）**：JNI 路径下不再有独立的 llama.cpp 二进制需要替换。
>
> 升级 MiBrain APK 即同时升级 `libllama.so` + `libggml.so`（两者已打入 APK 的
> `jniLibs/arm64-v8a/`）。如需更新 llama.cpp 版本，由开发者在构建时 bump 依赖并发布新 APK，
> 用户只需走 §9.1 流程。

### 9.4 升级模型
直接替换 `/data/user_de/0/com.mibrain/files/models/` 下的文件。无需重启，APK 会用新模型下次对话。

---

## 10. 常见问题速查

| 现象 | 原因 | 解决 |
|---|---|---|
| 打开 APK 提示"LlamaEngine 加载失败" | JNI 加载失败 / 模型路径错 / .so 缺失 | 检查 DE 区模型完整性，看 `install.log` 与 logcat |
| 说唤醒词没反应 | KWS 阈值太高 / 麦克风被拦 | 降阈值 / 装 Phantom Mic |
| 锁屏后唤醒失败 | MIUI 杀进程 / Direct Boot 未生效 | 重新做 §5 全部 9 步；确认 APK 声明 `directBootAware=true`（[D21](../DECISIONS.md)） |
| 回应很慢（>15s） | 模型太大 / 内存压力 | 已是默认 Qwen2.5-1.5B；**3B 在 8GB 设备必 OOM 不可切（[D30](../DECISIONS.md)）**，先看内存（关闭其他大内存 App），或调短 `keep_alive_minutes` |
| TTS 声音不自然 | VITS 模型质量 | 升级到 matcha-icefall-zh-baker 等，**但许可需复核**（[D22](../DECISIONS.md)，默认配置已是 Apache 2.0 友好） |
| 内存不足被杀 | 系统压力大 | 关闭其他大内存 App；切回 1.5B 默认 |
