# KSU 模块骨架

> 此目录是 KernelSU 模块源码骨架，**当前为空**，Phase 1 后填入实际脚本。

## 计划结构（Phase 1 完成后）

```
ksu_module/
├── module.prop             # 模块元信息
├── install.sh              # 架构检查 + 权限设置
├── post-fs-data.sh         # 创建目录 + chcon SELinux
# service.sh 已删除（[D7](../DECISIONS.md) 切回 JNI 后不再有 service.sh 启动推理服务；
# KSU 仅用 post-fs-data.sh 配置 appops + sepolicy.rule 放行）
├── uninstall.sh            # 清理进程（不删模型）
├── sepolicy.rule           # KSU SELinux 放行规则
└── system/etc/sysconfig/
    └── mibrain.xml         # 电池白名单
# libs/ 子目录已删除（不再打包 llama-server 二进制到 KSU 模块；
# .so 改由 APK jniLibs 携带，[D7](../DECISIONS.md)）
```

## 模块职责

KSU 模块**只做两件事**：

1. **设置 appops 权限**（让 APK 有 RECORD_AUDIO 权限，绕过 HyperOS 部分限制）
2. **加入电池白名单 + 关闭应用智能休眠**（[D31](../DECISIONS.md)，防止 HyperOS 3 锁屏杀进程）

**不再启动 llama-server**（[D7](../DECISIONS.md) 切回 JNI，推理在 APK 内完成）

**不打包**：
- 模型文件（用户自己下载）
- APK（独立安装）
- Phantom Mic（用户自行从 LSPosed 安装）

## 生成 zip

```bash
# Phase 1 完成后会有
./ksu_module/build.sh
```
