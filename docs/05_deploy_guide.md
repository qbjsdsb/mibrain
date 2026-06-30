# 部署指南（用户向）

> 本文档面向最终用户：拿到 MiBrain 后如何在红米 K50 Ultra 上跑起来。

---

## 0. 前置条件检查

在开始之前，确认你的设备满足：

| 项 | 要求 | 验证方式 |
|---|---|---|
| 设备 | 红米 K50 Ultra（骁龙 8+ Gen 1） | 设置 → 我的设备 |
| 系统 | HyperOS 1+（Android 12+） | 设置 → 我的设备 → MIUI 版本 |
| Root | KernelSU 已装且工作 | KSU Manager 应用可打开 |
| Zygisk | ZygiskNext 已装且启用 | LSPosed 显示"Zygisk 已注入" |
| LSPosed | LSPosed 已装且启用 | LSPosed Manager 显示框架激活 |
| 存储 | 至少 5GB 可用空间 | 文件管理器查看 |
| 网络 | 首次配置需联网下载 ~2.4GB 模型 | - |

不满足任何一项 → 先解决再继续。

---

## 1. 下载所需文件

### 1.1 MiBrain 仓库代码

```bash
# 在电脑上
git clone https://github.com/qbjsdsb/mibrain.git
```

### 1.2 二进制依赖

| 文件 | 下载地址 | 大小 | 用途 |
|---|---|---|---|
| llama-server (android-arm64) | https://github.com/ggml-org/llama.cpp/releases/download/b9844/llama-b9844-bin-android-arm64.tar.gz | ~30MB | LLM 推理服务 |
| sherpa-onnx AAR | https://github.com/k2-fsa/sherpa-onnx/releases/download/v1.13.3/sherpa-onnx-1.13.3.aar | ~54MB | APK 用 |

### 1.3 模型文件

| 文件 | 下载地址 | 大小 | 用途 |
|---|---|---|---|
| Qwen2.5-3B-Instruct Q4_K_M | https://huggingface.co/Qwen/Qwen2.5-3B-Instruct-GGUF/resolve/main/qwen2.5-3b-instruct-q4_k_m.gguf | ~2GB | 默认对话模型 |
| paraformer 流式 ASR | https://github.com/k2-fsa/sherpa-onnx/releases/download/asr-models/sherpa-onnx-streaming-paraformer-bilingual-zh-en.tar.bz2 | ~250MB | 语音识别 |
| VITS aishell3 TTS | https://huggingface.co/k2-fsa/sherpa-onnx/resolve/main/tts-models/vits-icefall-zh-aishell3.tar.bz2 | ~150MB | 中文 TTS |
| openWakeWord hey_jarvis | https://huggingface.co/datasets/cscripka/openWakeWord/resolve/main/hey_jarvis.onnx | ~10MB | 唤醒词（MVP 用） |

合计约 2.4GB。

### 1.4 SHA256 校验

下载完后**必须校验 SHA256**，避免下错版本或损坏：

```bash
# 期望值（Phase 1 release 时填入，目前是占位）
sha256sum qwen2.5-3b-instruct-q4_k_m.gguf
# 期望: <Phase 1 release 时公布>
```

校验失败的文件不要用，重新下载。

---

## 2. 准备手机存储结构

通过 adb shell 或文件管理器：

```bash
mkdir -p /sdcard/MiBrain/models
mkdir -p /sdcard/MiBrain/models/paraformer-zh
mkdir -p /sdcard/MiBrain/models/vits-aishell3

# 把模型文件按如下结构放到手机：
/sdcard/MiBrain/
├── models/
│   ├── qwen2.5-3b-instruct-q4_k_m.gguf
│   ├── hey_jarvis.onnx
│   ├── paraformer-zh/
│   │   ├── encoder.int8.onnx
│   │   ├── decoder.int8.onnx
│   │   └── tokens.txt
│   └── vits-aishell3/
│       ├── model.onnx
│       ├── tokens.txt
│       └── lexicon.txt
└── config.json   (APK 首次启动时自动生成)
```

ASR/TTS 模型解压后路径以官方包内结构为准。

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

# 检查 llama-server 是否在跑
curl http://127.0.0.1:8080/health
# 期望: {"status": "ok"}

# 检查日志
cat /data/adb/mibrain/llama-server.log | tail -20
```

如果 `/health` 不通，看 [07_troubleshooting.md](./07_troubleshooting.md)。

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
3. 进入"模型设置"，确认路径自动检测正确
4. 点"测试连接"，应该显示"llama-server 已就绪"
5. 启用前台服务

---

## 5. 关键：MIUI/HyperOS 系统设置白名单（必做）

**这一步漏一个都会导致锁屏后无法唤醒**。在 HyperOS 上做完以下 8 步：

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

### 验证白名单生效
锁屏 5 分钟后，adb logcat 应该还能看到 MiBrain ForegroundService 在跑：
```bash
adb logcat | grep MiBrain
```

---

## 6. 安装 LSPosed Phantom Mic 模块

详见 [06_lspoded_setup.md](./06_lspoded_setup.md)。

简版步骤：
1. 下载 https://github.com/Xposed-Modules-Repo/tn.amin.phantom_mic/releases/download/3-2.0/PhantomMic-2.0.apk
2. `adb install PhantomMic-2.0.apk`
3. 打开 LSPosed Manager → 模块 → 启用 Phantom Mic
4. 在 Phantom Mic 范围里勾选 MiBrain
5. 重启手机

---

## 7. 验证完整链路

### 7.1 测试对话
1. 打开 MiBrain
2. 点"开始监听"
3. 说"hey jarvis"
4. 听到回应"嗯？"
5. 说"今天天气"
6. 听到 AI 回答

### 7.2 测试锁屏唤醒
1. 锁屏
2. 等 10 秒
3. 说"hey jarvis"
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
LSPosed → 禁用 → 卸载 APK

### 8.4 清理模型（可选）
```bash
adb shell
rm -rf /sdcard/MiBrain
```

---

## 9. 升级

### 9.1 升级 MiBrain APK
直接 `adb install -r` 覆盖安装，配置保留。

### 9.2 升级 KSU 模块
KSU Manager → 模块 → 从存储安装新版 zip → 重启

### 9.3 升级 llama.cpp 二进制
下载新版 android-arm64 包，替换 `/data/adb/modules/mibrain/libs/llama-server`，重启。

### 9.4 升级模型
直接替换 `/sdcard/MiBrain/models/` 下的文件。无需重启，APK 会用新模型下次对话。

---

## 10. 常见问题速查

| 现象 | 原因 | 解决 |
|---|---|---|
| 打开 APK 提示"llama-server 未就绪" | KSU 模块没启动 / 模型路径错 | 检查 `/health`，看 `install.log` |
| 说"hey jarvis"没反应 | 唤醒词阈值太高 / 麦克风被拦 | 降阈值 / 装 Phantom Mic |
| 锁屏后唤醒失败 | MIUI 杀进程 | 重新做 §5 全部 8 步 |
| 回应很慢（>15s） | 模型太大 / 内存压力 | 改用 Qwen2.5-1.5B |
| TTS 声音不自然 | VITS 模型质量 | 升级到 matcha-icefall-zh-baker |
| 内存不足被杀 | 系统压力大 | 关闭其他大内存 App |
