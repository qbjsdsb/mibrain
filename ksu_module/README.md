# KSU 模块骨架

> 此目录是 KernelSU 模块源码骨架，**当前为空**，Phase 1 后填入实际脚本。

## 计划结构（Phase 1 完成后）

```
ksu_module/
├── module.prop             # 模块元信息
├── install.sh              # 架构检查 + 权限设置
├── post-fs-data.sh         # 创建目录 + chcon SELinux
├── service.sh              # 启动 llama-server 二进制 + watchdog
├── uninstall.sh            # 清理进程（不删模型）
├── sepolicy.rule           # KSU SELinux 放行规则
├── system/etc/sysconfig/
│   └── mibrain.xml         # 电池白名单
└── libs/
    ├── llama-server        # llama.cpp android-arm64 二进制
    └── (whisper.cpp, piper 可选，后续阶段)
```

## 模块职责

KSU 模块**只做三件事**：

1. **启动 llama-server 二进制**（监听 127.0.0.1:8080，OpenAI 兼容 API）
2. **设置 appops 权限**（让 APK 有 RECORD_AUDIO 权限，绕过 MIUI 部分限制）
3. **加入电池白名单**（防止 MIUI 锁屏杀进程）

**不打包**：
- 模型文件（用户自己下载）
- APK（独立安装）
- Phantom Mic（用户自行从 LSPosed 安装）

## 生成 zip

```bash
# Phase 1 完成后会有
./ksu_module/build.sh
```
