# 故障排查指南

> 本文档列了 MiBrain 在红米 K50 Ultra + HyperOS 上常见的故障与排查路径。

---

## 1. 故障分类

按故障层分：

| 层 | 典型现象 | 排查入口 |
|---|---|---|
| KSU 模块层 | 模块未启动 / 二进制缺失 / SELinux 拒绝 | §2 |
| llama-server 层 | HTTP 不通 / 模型加载失败 / OOM | §3 |
| APK 层 | 启动崩溃 / 权限缺失 / 状态机异常 | §4 |
| 录音层 | 锁屏后哑 / 麦克风被占 | §5 |
| 模型层 | 识别差 / TTS 不自然 | §6 |
| 系统层 | MIUI 杀进程 / 内存压力 | §7 |

---

## 2. KSU 模块层故障

### 2.1 现象：模块显示已启用但 llama-server 没起

**排查步骤**：

```bash
# 1. 检查模块目录
adb shell
su
ls /data/adb/modules/mibrain/
# 期望: module.prop, service.sh, libs/, ...

# 2. 看 service.sh 是否被执行
cat /data/adb/modules/mibrain/install.log
# 期望: 有 "[MiBrain] ..." 日志

# 3. 检查二进制是否在
ls /data/adb/modules/mibrain/libs/llama-server
# 期望: 文件存在

# 4. 检查二进制可执行
/data/adb/modules/mibrain/libs/llama-server --version
# 期望: 输出版本号

# 5. 检查 SELinux 上下文
ls -Z /data/adb/modules/mibrain/libs/llama-server
# 期望: u:object_r:system_file:s0
```

**常见原因**：

| 原因 | 解决 |
|---|---|
| 二进制架构不对（下了 x86 版） | 重新下 arm64-v8a 版本 |
| 二进制没执行权限 | `chmod 755 llama-server` |
| SELinux 拒绝 | 在 sepolicy.rule 加规则 |
| 模型路径不对 | 检查 `/sdcard/MiBrain/models/` |

### 2.2 现象：service.sh 报错

```bash
cat /data/adb/mibrain/llama-server.log
```

常见错误：
- `failed to load model: invalid file magic` → 模型文件损坏，重新下载
- `out of memory` → 系统内存不足，关其他 App
- `bind: address already in use` → 8080 被占，改 config.json 的 port

---

## 3. llama-server 层故障

### 3.1 现象：HTTP 503 / 模型未加载

```bash
curl http://127.0.0.1:8080/health
# 503 表示模型还在加载或加载失败
```

排查：
```bash
# 检查模型加载日志
tail -50 /data/adb/mibrain/llama-server.log

# 检查模型文件
ls -la /sdcard/MiBrain/models/
# 期望 qwen2.5-3b-instruct-q4_k_m.gguf 存在且 >1.5GB

# 校验 SHA256
sha256sum /sdcard/MiBrain/models/qwen2.5-3b-instruct-q4_k_m.gguf
```

### 3.2 现象：CPU 100% 但生成慢

可能原因：
- 模型用了 Q8 而非 Q4（误下版本）
- CPU 核心没充分利用（threads=4 没生效）
- 设备过热降频

解决：
```bash
# 看模型加载时的参数
grep "n_threads" /data/adb/mibrain/llama-server.log
# 期望 4

# 看温度
cat /sys/class/thermal/thermal_zone*/temp 2>/dev/null | head -5
# >65 表示过热
```

---

## 4. APK 层故障

### 4.1 现象：APK 启动闪退

```bash
adb logcat | grep -iE "mibrain|AndroidRuntime"
```

常见错误：
- `UnsatisfiedLinkError: libsherpa-onnx-jni.so` → AAR 没正确集成，看 `app/jniLibs/arm64-v8a/`
- `SecurityException: RECORD_AUDIO` → 用户没授录音权限
- `IllegalStateException: foregroundServiceType` → Manifest 缺 `foregroundServiceType="microphone"`

### 4.2 现象：唤醒词检测不工作

```bash
adb logcat | grep -i wakeword
```

排查：
1. 检查唤醒词模型路径：`/sdcard/MiBrain/models/hey_jarvis.onnx`
2. 检查唤醒阈值：config.json 里的 `wakeword.threshold`
3. 调高阈值太严 → 调低到 0.3
4. 调低阈值太松 → 调高到 0.7

### 4.3 现象：状态机卡在某个状态

看 APK 内部状态日志：
```bash
adb logcat | grep -i "ConversationState"
```

期望状态转移：`IDLE → LISTENING → THINKING → SPEAKING → COOLDOWN → IDLE`

如果卡在 `THINKING` 不动 → llama-server 没响应
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
1. 重做 [05_deploy_guide.md §5](./05_deploy_guide.md) 全部 8 步白名单配置
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

排查：
```bash
# 录一段测试音频
adb shell
# 在 APK 里加调试模式，dump 原始 PCM
# 用 ffmpeg 转换后用 PC 工具分析
```

### 6.2 现象：TTS 声音机械

VITS aishell3 模型质量一般。升级到 matcha-icefall-zh-baker + vocos：
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
- 把 Qwen2.5-3B 换成 1.5B（省 1GB）
- 调短 `keep_alive_minutes`（5→3）

### 7.2 现象：设备严重发热

8+ Gen 1 跑 3B 模型 + ASR + TTS 会发热。

```bash
cat /sys/class/thermal/thermal_zone*/temp
```

如果 >75℃，先停用一段时间。可考虑：
- 加散热背夹
- 减少 threads=2
- 改用 1.5B 模型

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
adb shell cat /data/adb/mibrain/llama-server.log > llama-server.log
adb shell dmesg > dmesg.log
```

提交 Issue 时附上这三个日志。

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
- llama-server.log
- MiBrain APK 版本
- Phantom Mic 启用状态
- 已尝试的排查步骤

详见 [.github/ISSUE_TEMPLATE/bug_report.md](../.github/ISSUE_TEMPLATE/bug_report.md)。

---

## 10. 已知限制

| 限制 | 原因 | 缓解 |
|---|---|---|
| 锁屏唤醒在 HyperOS 2 上不保证 | Phantom Mic 兼容性未验证 | Phase 4 真机测试 |
| 8GB 内存峰值紧张 | 硬件限制 | 用 1.5B 模型 |
| 中文唤醒词需自训 | openWakeWord 无现成 | MVP 用 hey_jarvis |
| TTS 不够自然 | VITS aishell3 质量 | 升级到 matcha+vocos |
| 无 RAG | MVP 范围外 | 装 ToolNeuron 补 RAG |
