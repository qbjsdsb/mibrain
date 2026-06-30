# 设计文档目录

## 文档清单

| 文件 | 内容 | 状态 |
|---|---|---|
| [00_design_overview.md](./00_design_overview.md) | 完整设计文档（架构、组件、路线图） | ✅ |
| [01_feasibility_verification.md](./01_feasibility_verification.md) | 可行性验证报告（15/15 依赖项实测可达，含 3 项已弃用） | ✅ |
| [02_second_review.md](./02_second_review.md) | 第二轮严谨审视（5 个新发现 + 修订） | ✅ |
| [03_architecture_detail.md](./03_architecture_detail.md) | 详细架构与时序图（进程拓扑、协议、状态机） | ✅ |
| 04_build_guide.md | 编译指南 | ⏳ Phase 1 |
| [05_deploy_guide.md](./05_deploy_guide.md) | 部署指南（用户向，含 8 步白名单） | ✅ |
| [06_lspoded_setup.md](./06_lspoded_setup.md) | LSPosed 配置指南（Phantom Mic 详解） | ✅ |
| [07_troubleshooting.md](./07_troubleshooting.md) | 故障排查（7 大类故障 + 排查流程） | ✅ |
| 08_performance_bench.md | 性能基准 | ⏳ Phase 5 |
| [09_phase6_network_tools_design.md](./09_phase6_network_tools_design.md) | Phase 6 联网工具调用设计（开关 + 4 工具） | ✅ 草案 |
| [10_phase7_phone_control_design.md](./10_phase7_phone_control_design.md) | Phase 7 手机控制类工具设计（4 工具） | ✅ 草案 |
| [11_phase8_platform_design.md](./11_phase8_platform_design.md) | Phase 8 平台化能力设计（API/Widget/MCP） | ✅ 草案 |
| [12_phase9_multimodal_design.md](./12_phase9_multimodal_design.md) | Phase 9 多模态设计（OCR + 看图对话） | ✅ 草案 |
| [13_phase10_ux_enhancements_design.md](./13_phase10_ux_enhancements_design.md) | Phase 10 用户体验增强（多用户/儿童/i18n/a11y/音频路由/全局暂停） | ✅ 草案 |

## 当前阶段

**Phase 0：设计冻结**

所有设计决策已确认（详见 [../DECISIONS.md](../DECISIONS.md)）。等仓库初始化完成后进入 Phase 1。

## 阅读顺序

新贡献者请按此顺序阅读：

1. [README.md](../README.md) — 项目概览
2. [00_design_overview.md](./00_design_overview.md) — 完整设计
3. [01_feasibility_verification.md](./01_feasibility_verification.md) — 验证报告
4. [02_second_review.md](./02_second_review.md) — 第二轮修订
5. [03_architecture_detail.md](./03_architecture_detail.md) — 详细架构
6. [05_deploy_guide.md](./05_deploy_guide.md) — 部署指南
7. [06_lspoded_setup.md](./06_lspoded_setup.md) — LSPosed 配置
8. [07_troubleshooting.md](./07_troubleshooting.md) — 故障排查
9. [09_phase6_network_tools_design.md](./09_phase6_network_tools_design.md) — Phase 6 联网工具调用设计
10. [10_phase7_phone_control_design.md](./10_phase7_phone_control_design.md) — Phase 7 手机控制设计
11. [11_phase8_platform_design.md](./11_phase8_platform_design.md) — Phase 8 平台化设计
12. [12_phase9_multimodal_design.md](./12_phase9_multimodal_design.md) — Phase 9 多模态设计
13. [../DECISIONS.md](../DECISIONS.md) — 决策清单
