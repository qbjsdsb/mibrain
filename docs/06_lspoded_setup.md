# LSPosed 配置指南

> 本文档说明如何在 LSPosed 里启用 Phantom Mic 模块，让 MiBrain 在 MIUI/HyperOS 后台也能稳定录音。

> **2026-06-30 修订说明**（第三轮 web 核实 [F1](./14_feasibility_recheck_and_plan.md) + [F2](./14_feasibility_recheck_and_plan.md)）：
> - **LSPosed 框架升级为 LSPosed Vector**（[D9](../DECISIONS.md)）：原 LSPosed 项目自 2024 年起活跃度下降，社区接力维护分支 **LSPosed Vector** 是当前唯一活跃维护版本
>   - v2.0.1：2026-04-21 发布
>   - v2.0.2：2026-05-05 发布
>   - **v2.0.3-7716：2026-05-20 发布**（本次设计冻结时最新）
>   - 兼容性：Android 8.1 - Android 17 Beta 3
> - **HyperOS 升级为 HyperOS 3**（[D6](../DECISIONS.md)）：HyperOS 3 国行版已对 K50U 推送 OS3.0.1.0.VLFCNXM（2026-01-20 OTA）/ OS3.0.2.0.VLFCNXM（2026-04-13 Fastboot），基于 Android 15
> - Phantom Mic 风险升级为高（[D14](../DECISIONS.md)）：v2.0 自 2024-07 发布至本次设计冻结已近 2 年未更新，**LSPosed Vector 框架升级 ≠ Phantom Mic 模块兼容性升级**，仍需 Phase 1 启动前先去 [上游仓库](https://github.com/Xposed-Modules-Repo/tn.amin.phantom_mic/releases) 复查活跃度
> - 唤醒词改 sherpa-onnx KWS（[D23](../DECISIONS.md)）：弃用 openWakeWord，本文件 §5.3 测试用"hey jarvis"仍有效

---

## 1. 前置条件

- ✅ KernelSU 已装且启用
- ✅ ZygiskNext 已装且在 KSU 里启用
- ✅ **LSPosed Vector v2.0.3-7716**（[D9](../DECISIONS.md)）已装且框架激活

验证 LSPosed Vector 是否正常：

```bash
adb shell
su
ls /data/adb/lspd
# 应该有 config.bin, framework.jar 等文件
```

或打开 LSPosed Vector Manager，主界面应该显示"激活"，并在关于页看到版本号 v2.0.3-7716。

⚠️ **重要**（[F1](./14_feasibility_recheck_and_plan.md)）：原 LSPosed 项目（`LSPosed/LSPosed`）自 2024 年起活跃度下降，**不要再装原版 LSPosed**。本次设计冻结时唯一活跃维护版本是 LSPosed Vector（fork 自 LSPosed），下载入口见 [LSPosed Vector Releases](https://github.com/LSPosed/LSPosed/releases)（占位链接，实际下载地址在 Phase 1 实施前复查上游确认）。

⚠️ Phantom Mic v2.0 自 2024-07 发布至本次设计冻结（2026-06-30）已近 2 年未更新，上游活跃度存疑。Phase 1 启动前先去 [上游仓库](https://github.com/Xposed-Modules-Repo/tn.amin.phantom_mic/releases) 复查是否有新版本或 HyperOS 3 / Android 15 兼容性反馈（[D14](../DECISIONS.md)）。

---

## 2. Phantom Mic 是什么

[Phantom Mic](https://github.com/Xposed-Modules-Repo/tn.amin.phantom_mic) 是 LSPosed 官方仓库的模块，**hook Android native 层的 `AudioRecord.cpp`**，让指定 App 即使在后台也能正常录音。

它解决的核心问题：
- MIUI 锁屏后 `AudioRecord.read()` 返回 0 字节
- 即使有 `RECORD_AUDIO` 权限 + 前台服务 + WakeLock 也无效
- 错误日志：`AF::OpRecordAudioMonitor: OP_RECORD_AUDIO missing`

---

## 3. 安装 Phantom Mic

### 3.1 下载 APK

```bash
# 在电脑上
wget https://github.com/Xposed-Modules-Repo/tn.amin.phantom_mic/releases/download/3-2.0/PhantomMic-2.0.apk
```

⚠️ 如果上游已发布新版本（v3+），优先用新版本。Phase 1 启动前先复查（[D14](../DECISIONS.md)）。

SHA256 校验（Phase 4 真机验证后填入）：

### 3.2 安装到手机

```bash
adb install PhantomMic-2.0.apk
```

---

## 4. 在 LSPosed 启用

### 4.1 启用模块

1. 打开 LSPosed Manager
2. 主页 → 模块
3. 找到 **Phantom Mic**
4. 点右侧开关，启用模块

### 4.2 设置作用域（关键）

这一步**最关键**：

1. 在 Phantom Mic 卡片上点"作用域"或齿轮图标
2. 在应用列表里找到 **MiBrain**
3. **勾选**（开关打开）
4. 上方保存

### 4.3 配置 Phantom Mic 选项（可选）

打开 Phantom Mic 应用（桌面有图标）：

| 选项 | 推荐值 | 说明 |
|---|---|---|
| 启用模块 | ✅ | 必须开 |
| 作用域已勾选 | ✅ | 见 §4.2 |
| 录音文件夹 | 留空 | 留空 = 用真实麦克风 |
| phantom.txt | 留空 | 留空 = 正常麦克风输入 |
| 调试日志 | 关 | 除非排查问题 |

### 4.4 重启

LSPosed 模块需要重启才生效：

```bash
adb reboot
```

或在手机上长按电源 → 重启。

---

## 5. 验证 Phantom Mic 工作

### 5.1 检查 LSPosed 状态

打开 LSPosed Manager → 模块 → Phantom Mic：
- 模块状态：**已启用**
- 作用域：包含 MiBrain

### 5.2 检查 hook 是否生效

```bash
adb logcat | grep -i phantom
# 启动 MiBrain 后应该有 Phantom Mic 注入日志
```

### 5.3 实测：锁屏录音

1. 打开 MiBrain，启用监听
2. 锁屏
3. 等 30 秒
4. 说"hey jarvis"（唤醒词由 sherpa-onnx KWS 检测，[D23](../DECISIONS.md)；MVP 仍用英文 hey_jarvis，[D2](../DECISIONS.md)）
5. 应该能正常唤醒

如果唤不醒：
- adb logcat 看 MiBrain 是否在跑（ForegroundService 应该还在）
- 看 AudioRecord.read() 是否还返回 0
- 参考 [07_troubleshooting.md](./07_troubleshooting.md)

---

## 6. Phantom Mic 不工作怎么办

### 6.1 HyperOS 3 (Android 15) 兼容性问题

⚠️ **风险升级**（[D14](../DECISIONS.md)）：Phantom Mic v2.0 自 2024-07 发布至本次设计冻结已近 2 年未更新，HyperOS 3（Android 15）兼容性未验证，风险从"中"升级为"高"。

**LSPosed Vector 框架 ≠ Phantom Mic 模块**：LSPosed Vector v2.0.3 已支持 Android 8.1-17 Beta 3，框架本身可在 HyperOS 3 上激活。但 Phantom Mic 是 LSPosed 模块，模块内部依赖的 LSPosed API 版本可能滞后，仍可能失效。

进入 Phase 1 启动前先复查：
1. 去 https://github.com/Xposed-Modules-Repo/tn.amin.phantom_mic/releases 看是否有新版本
2. 看 Issues 区是否有 HyperOS 3 / Android 15 兼容性反馈
3. 同时确认 LSPosed Vector v2.0.3 在 HyperOS 3 K50U 上是否激活（无框架 = 模块也无用）
4. 如果上游确认停滞，按以下方案 B/C 之一替代

如果 Phase 4 真机验证 Phantom Mic 在 HyperOS 3 上无效：

**方案 A：降级到 HyperOS 1 或 HyperOS 2（不推荐）**
- HyperOS 1/2 基于 Android 12-15
- 但降级会丢失系统新功能，且 HyperOS 3 已对 K50U 推送，不应再退

**方案 B：用 appops 强制 + 双触发兜底**
KSU 模块的 `post-fs-data.sh` 自动执行：
```bash
appops set <mibrain_uid> RECORD_AUDIO allow
appops set <mibrain_uid> OP_RECORD_AUDIO allow
```
APK 里加一个桌面快捷方式兜底（双击电源键 → 触发录音）。

**方案 C：换用其他 LSPosed 录音 hook 模块**
候选：
- [XAudioCapture](https://github.com/wzhy90/XAudioCapture)（主要 hook 播放录音，非麦克风）
- [WhatsMicFix](https://github.com/D4vRAM369/WhatsMicFix-LSPosed)（专为 WhatsApp，可改造）

### 6.2 模块冲突

如果同时装了多个 LSPosed 录音 hook，可能冲突。**只装一个**。

### 6.3 ZygiskNext 没启用

LSPosed 必须依赖 ZygiskNext 提供 Zygisk 运行时。

```bash
adb shell
su
ls /data/adb/modules/zygisk_next
# 应该有 module.prop, system文件夹等
```

如果没装，参考 [ZygiskNext 安装指南](https://github.com/Dr-TSNG/ZygiskNext)。

---

## 7. Phantom Mic 的工作原理（技术细节）

Phantom Mic hook 的是 Android native 层的 `AudioRecord.cpp`，具体方法：

```
frameworks/av/media/libaudiohal/.../AudioRecord.cpp
  - read()
  - getMinFrameCount()
  - set(...)
```

通过 LSPosed 的 `Native API`（Zygisk 提供），在目标 App 启动时注入 hook 代码：

1. App 调 `AudioRecord.read(buf, size)`
2. Phantom Mic 拦截
3. Phantom Mic 直接调底层 `IAudioRecord.read()`（绕过 MIUI 的 AppOps 检查）
4. 把数据返回给 App
5. App 拿到真实音频数据

这套机制对 App 透明，不需要改 App 代码。

---

## 8. 配置截图（待 Phase 4 补充）

LSPosed 配置界面的截图会在 Phase 4 真机测试时补充到 `docs/assets/` 目录。

---

## 9. 卸载 Phantom Mic

1. LSPosed Manager → 模块 → Phantom Mic → 关闭
2. 重启
3. `adb uninstall tn.amin.phantom_mic`

卸载后 MiBrain 在 HyperOS 2 上的后台录音可能失效，回到 §6 的备选方案。

---

## 10. 风险提示

| 风险 | 影响 | 缓解 |
|---|---|---|
| Phantom Mic 是 root 级 hook，安全性敏感 | 中 | 来自 LSPosed 官方仓库，可信 |
| HyperOS 升级后 hook 失效 | 中 | 升级前先验证 |
| LSPosed 框架本身被检测 | 低 | 一般 App 不检测 |
| 同时装多个录音 hook 冲突 | 中 | 只装一个 |
| Phantom Mic 上游停滞 / HyperOS 2 不兼容 | 高 | v2.0 已近 2 年未更新（[D14](../DECISIONS.md)）；Phase 1 启动前先复查上游活跃度；Phase 4 真机验证后准备 fallback（appops + 双触发兜底） |
