# 故障排查指南

> 本文档列了 MiBrain 在红米 K50 Ultra + HyperOS 上常见的故障与排查路径。

> **2026-06-30 修订说明**（深度检查后第二轮调整）：
> - 推理后端从 llama-server HTTP 切回 JNI（[D7](../DECISIONS.md) + [X7](../DECISIONS.md)）：原 §2/§3 的 service.sh / llama-server.log / curl /health 检查不再适用
> - 模型路径改 DE 加密区（[D21](../DECISIONS.md)）：原 /sdcard/MiBrain/models/ 改为 /data/user_de/0/com.mibrain/files/models/
> - 默认模型从 3B 改 1.5B（[D1](../DECISIONS.md)）
> - ASR/TTS 模型换 Apache 2.0 许可（[D22](../DECISIONS.md)）
> - 唤醒词改 sherpa-onnx KWS（[D23](../DECISIONS.md)）

---

## 1. 故障分类

按故障层分：

| 层 | 典型现象 | 排查入口 |
|---|---|---|
| KSU 模块层 | 模块未启动 / appops 未生效 / SELinux 拒绝 | §2 |
| JNI 推理层 | LlamaEngine 加载失败 / OOM / JNI 崩溃 | §3 |
| APK 层 | 启动崩溃 / 权限缺失 / 状态机异常 | §4 |
| 录音层 | 锁屏后哑 / 麦克风被占 | §5 |
| 模型层 | 识别差 / TTS 不自然 | §6 |
| 系统层 | MIUI 杀进程 / 内存压力 | §7 |

---

## 2. KSU 模块层故障

### 2.1 现象：模块显示已启用但 appops 未生效

**排查步骤**：

```bash
# 1. 检查模块目录
adb shell
su
ls /data/adb/modules/mibrain/
# 期望: module.prop, post-fs-data.sh, sepolicy.rule, ...

# 2. 看 post-fs-data.sh 是否被执行
cat /data/adb/mibrain/install.log
# 期望: 有 "[MiBrain] appops set ..." 日志

# 3. 检查 appops 是否生效
pm list packages -U com.mibrain | awk '{print $2}' | cut -d/ -f2
# 输出 uid=10234
appops get 10234 RECORD_AUDIO
# 期望: RECORD_AUDIO: allow; time=...
```

### 2.2 现象：post-fs-data.sh 报错

```bash
cat /data/adb/mibrain/install.log
```

常见错误：
- `appops set` 失败 → KSU root 未生效，检查 KSU Manager
- `chcon` 失败 → sepolicy.rule 未匹配 HyperOS 3，参考 [06_lspoded_setup.md §6](./06_lspoded_setup.md)

---

## 3. JNI 推理层故障

### 3.1 现象：LlamaEngine 加载失败 / OOM

```bash
# 检查 LLM 模型文件
adb shell
ls -la /data/user_de/0/com.mibrain/files/models/qwen2.5-1.5b-instruct-q4_k_m.gguf
# 期望: 文件存在且 >900MB（1.5B 模型，[D1](../DECISIONS.md)）

# 校验 SHA256
sha256sum /data/user_de/0/com.mibrain/files/models/qwen2.5-1.5b-instruct-q4_k_m.gguf

# 看 APK 日志
adb logcat | grep -iE "LlamaEngine|llama"
# 期望: "loadModel success"
# 如果 OOM: "llama_decode: out of memory"
```

### 3.2 现象：CPU 100% 但生成慢

可能原因：
- 模型用了 Q8 而非 Q4（误下版本）
- CPU 核心没充分利用（threads=4 没生效）
- 设备过热降频

解决：
```bash
# 看 LlamaEngine 启动参数
adb logcat -d | grep -iE "LlamaEngine|n_threads"
# 期望 n_threads=4

# 看温度
cat /sys/class/thermal/thermal_zone*/temp 2>/dev/null | head -5
# >65 表示过热
```

补充说明：
- 已默认 1.5B；若用 3B 质量优先模式 OOM 则切回 1.5B（[D1](../DECISIONS.md)）

---

## 4. APK 层故障

### 4.1 现象：APK 启动闪退

```bash
adb logcat | grep -iE "mibrain|AndroidRuntime"
```

常见错误：
- `UnsatisfiedLinkError: libsherpa-onnx-jni.so` → AAR 没正确集成，看 `app/jniLibs/arm64-v8a/`
- `UnsatisfiedLinkError: libllama.so` → JNI wrapper 编译时未正确链接 libllama.so，检查 `app/src/main/jni/CMakeLists.txt`（参考 [llama.android 官方模块](https://github.com/ggml-org/llama.cpp/tree/master/examples/llama.android)）
- `directBootAware` 相关错误 → 检查 AndroidManifest.xml 是否声明（[D21](../DECISIONS.md)）
- `SecurityException: RECORD_AUDIO` → 用户没授录音权限
- `IllegalStateException: foregroundServiceType` → Manifest 缺 `foregroundServiceType="microphone"`

### 4.2 现象：唤醒词检测不工作

```bash
adb logcat | grep -iE "wakeword|SherpaKws"
```

> 唤醒词引擎改用 sherpa-onnx KWS（[D23](../DECISIONS.md)），日志关键字为 "SherpaKws" 而非 "wakeword"。

排查：
1. 检查唤醒词模型路径：`/data/user_de/0/com.mibrain/files/models/hey_jarvis.onnx`（[D21](../DECISIONS.md)）
2. 检查唤醒阈值：config.json 里的 `wakeword.threshold`
3. 调高阈值太严 → 调低到 0.3
4. 调低阈值太松 → 调高到 0.7

### 4.3 现象：状态机卡在某个状态

看 APK 内部状态日志：
```bash
adb logcat | grep -i "ConversationState"
```

期望状态转移：`IDLE → LISTENING → THINKING → SPEAKING → COOLDOWN → IDLE`

如果卡在 `THINKING` 不动 → LlamaEngine 没响应（JNI 卡住或模型未加载）
如果卡在 `SPEAKING` 不动 → AudioTrack 异常

---

## 5. 录音层故障（最常见）

### 5.1 现象：锁屏后唤醒失败

**90% 是这个原因**：MIUI 锁屏杀进程。

**排查流程**：

```bash
# 1. 锁屏前先看 MiBrain ForegroundService 是否在跑
adb shell
ps -A | grep mibrain
# 期望看到进程

# 2. 锁屏 30 秒后
ps -A | grep mibrain
# 如果进程没了 → 被杀了
```

如果进程被杀：
1. 重做 [05_deploy_guide.md §5](./05_deploy_guide.md) 全部 9 步白名单配置
2. 检查 Phantom Mic 是否启用：LSPosed Manager → Phantom Mic → 状态
3. 看是否勾选了 MiBrain：LSPosed → Phantom Mic → 作用域

### 5.2 现象：录音全是静音

```bash
adb logcat | grep -i "AudioRecord\|PhantomMic"
# 期望: Phantom Mic 注入日志
# 如果没有 → Phantom Mic 未生效
```

如果 Phantom Mic 没生效：
- 检查 ZygiskNext 是否启用
- 检查 LSPosed 是否激活
- 重启手机

### 5.3 现象：说话有回声

TTS 播放声音被麦克风拾起。检查状态机：
```bash
adb logcat | grep -i "ConversationState\|AudioFocus"
```

期望 `SPEAKING` 状态下不喂 ASR。如果状态对但还是有回声，可能是冷却时间太短，调长 `cooldown_ms` 到 1000ms。

---

## 6. 模型层故障

### 6.1 现象：ASR 识别率低

可能原因：
- 录音采样率不对（应为 16kHz mono int16）
- 模型选错（用了英文 ASR）
- 噪音环境

> ASR 模型改用 sherpa-onnx streaming-zipformer-bilingual-zh-en（[D22](../DECISIONS.md)），默认 Apache 2.0 许可。

排查：
```bash
# 录一段测试音频
adb shell
# 在 APK 里加调试模式，dump 原始 PCM
# 用 ffmpeg 转换后用 PC 工具分析
```

### 6.2 现象：TTS 声音机械

默认 TTS 模型为 sherpa-onnx-vits-zh-ll（社区贡献，许可未明确声明，[D22](../DECISIONS.md)；HF 卡未声明许可，分发 APK 存在风险）。如需更好自然度或规避许可风险，可升级到 matcha-icefall-zh-baker + vocos（matcha-icefall-zh-baker 已验证 Apache 2.0，[D22](../DECISIONS.md)）：
```
https://huggingface.co/csukuangfj/matcha-icefall-zh-baker
https://github.com/k2-fsa/sherpa-onnx/releases/download/vocoder-models/vocos-22khz-univ.onnx
```

### 6.3 现象：LLM 回答不相关

可能是 ctx_size 太小，上下文丢了。改 config.json：
```json
"ctx_size": 4096
```

但内存会多占 ~500MB。

---

## 7. 系统层故障

### 7.1 现象：内存压力触发 lmkd

```bash
adb shell
cat /proc/meminfo | head -5
# 看 MemAvailable，如果 <500MB 就危险

# 看 lmkd 是否杀过 MiBrain
dmesg | grep -i "lmkd\|lowmemorykiller"
```

解决：
- 关闭其他大内存 App（微信、抖音）
- 默认已是 1.5B；若用户切到 3B 质量优先模式则换回 1.5B（[D1](../DECISIONS.md)）
- 调短 `keep_alive_minutes`（5→3）

> 默认 1.5B 模型串行峰值 ~7.5GB（[03_architecture_detail.md §6](./03_architecture_detail.md)），留 0.5GB headroom（最坏叠加 7.89GB，紧张）。

### 7.2 现象：设备严重发热

8+ Gen 1 跑 1.5B 模型 + ASR + TTS 会发热，3B 模式发热更明显。

```bash
cat /sys/class/thermal/thermal_zone*/temp
```

如果 >75℃，先停用一段时间。可考虑：
- 加散热背夹
- 减少 threads=2
- 已是默认；若切了 3B 则切回 1.5B

### 7.3 现象：HyperOS 升级后失效

每次系统升级后，所有 hook 可能失效。重做：
1. 验证 ZygiskNext 是否还工作
2. 验证 LSPosed 是否还激活
3. 验证 Phantom Mic 是否还注入
4. 重新做白名单配置（路径可能变）

---

## 8. 通用排查工具

### 8.1 一键诊断脚本

```bash
adb shell
su
/data/adb/modules/mibrain/scripts/diagnose.sh
```

(Phase 4 时提供此脚本，输出诊断报告)

### 8.2 日志收集

```bash
adb logcat -v time > mibrain_logcat.log
adb shell cat /data/adb/mibrain/install.log > install.log
adb shell dmesg > dmesg.log
```

> APK 内部日志（LlamaEngine / Sherpa 模块）单独捞一份：
>
> ```bash
> adb logcat -v time | grep -iE "MiBrain|LlamaEngine|Sherpa" > mibrain_logcat.log
> ```

提交 Issue 时附上这些日志。

### 8.3 性能监控

```bash
adb shell
# 实时 CPU
top -m 10

# 实时内存
watch -n 1 free -m

# 实时温度
watch -n 1 "cat /sys/class/thermal/thermal_zone*/temp"
```

---

## 9. 何时该报 Issue

出现以下情况之一就报 Issue：

- 按本文档排查后仍解决不了
- 怀疑是 bug（崩溃 / 行为异常）
- 性能数据异常（明显低于预期）
- 兼容性问题（升级后失效）

提交时附：
- 设备信息（型号、系统版本、Root 方案）
- 完整 logcat
- install.log（KSU 模块安装日志）
- mibrain_logcat.log（APK 内部日志，含 LlamaEngine / Sherpa）
- MiBrain APK 版本
- Phantom Mic 启用状态
- 已尝试的排查步骤

详见 [.github/ISSUE_TEMPLATE/bug_report.md](../.github/ISSUE_TEMPLATE/bug_report.md)。

---

## 10. 已知限制

| 限制 | 原因 | 缓解 |
|---|---|---|
| 锁屏唤醒在 HyperOS 3 上不保证 | Phantom Mic 兼容性未验证 | Phase 4 真机测试 |
| 8GB 内存峰值紧张 | 硬件限制 | 用 1.5B 模型 |
| 中文唤醒词需自训 | sherpa-onnx KWS 默认英文，[D23](../DECISIONS.md) | MVP 用 hey_jarvis |
| TTS 不够自然 | VITS vits-zh-ll 质量（[D22](../DECISIONS.md)），可升级到 matcha+vocos（matcha-icefall-zh-baker 已验证 Apache 2.0） | 默认 vits-zh-ll，按需升级 |
| 无 RAG | MVP 范围外 | 装 ToolNeuron 补 RAG |
