# MiBrain 项目贡献指南

## 当前状态

项目处于**设计阶段**，尚未开始编码。在 Phase 1 启动前，不接受功能代码 PR。

欢迎的贡献类型：
- 设计文档改进（`docs/` 目录）
- 验证报告补充（在真机上测试某个组件并提交报告）
- Bug 报告（Issues）
- 文档翻译

## 开发约定（Phase 1 之后适用）

### 代码风格
- Kotlin: 遵循 [Kotlin Coding Conventions](https://kotlinlang.org/docs/coding-conventions.html)
- Shell: POSIX 兼容，使用 4 空格缩进
- 提交信息: Conventional Commits 规范

### 提交信息格式
```
<type>(<scope>): <subject>

<body>

<footer>
```

type: feat | fix | docs | style | refactor | test | chore | build | ci | perf

scope: app | ksu | daemon | docs | scripts

示例：
```
feat(app): 集成 sherpa-onnx 流式 ASR
fix(ksu): 修复 post-fs-data.sh 在 HyperOS 3 上的启动顺序
docs: 补充 LSPosed 配置说明
```

### 分支策略
- `main`: 稳定分支，受保护
- `develop`: 开发分支
- `feature/<scope>-<short-desc>`: 功能分支
- `fix/<scope>-<short-desc>`: 修复分支

### PR 流程
1. Fork 仓库
2. 创建 feature 分支
3. 提交 PR 到 `develop`
4. 至少 1 个 reviewer approve
5. CI 通过
6. Squash merge

## 真机测试报告提交格式

如果你在某台设备上测试了 MiBrain 的某个组件，请按此格式提交 Issue：

```
### 设备信息
- 型号: 红米 K50 Ultra
- 系统: HyperOS 3.0 (Android 15)
- Root: KernelSU v1.0 + ZygiskNext + LSPosed

### 测试内容
<测试了什么组件，做了什么操作>

### 结果
- 通过 / 部分通过 / 失败
- 性能数据（如 tok/s、内存占用）
- 详细日志

### 截图/录屏
<附件>
```

## 仓库结构参考

详见 [README.md](../README.md) 和 [docs/00_design_overview.md](./00_design_overview.md)。

## 联系方式

通过 GitHub Issues 提交问题，不提供其他联系方式。
