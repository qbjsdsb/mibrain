# MiBrain 开放决策清单

> 这些问题需要项目所有者回答后才能进入 Phase 1。  
> 已回答的标 ✅，未回答的标 ⏳。

## 待回答

### Q1. 默认对话模型
- 候选 A: Qwen2.5-3B-Instruct Q4_K_M (~2GB，质量更好)
- 候选 B: Qwen2.5-1.5B-Instruct Q4_K_M (~1GB，省内存)
- **建议**: A（8GB 内存能扛住，质量明显更好）
- 状态: ⏳ 待答

### Q2. 唤醒词策略
- 选项 A: MVP 用英文 "hey_jarvis"，Phase 3 再训中文
- 选项 B: MVP 不做唤醒词，用桌面快捷方式/双击电源键触发
- 选项 C: 现在就开始采集中文唤醒词样本（推迟 Phase 1）
- **建议**: B（MVP 先跑通主链路）
- 状态: ⏳ 待答

### Q3. 是否做 RAG
- 选项 A: MVP 不做，Phase 5 之后再说
- 选项 B: MVP 集成最简 RAG（用户文档问答）
- **建议**: A（ToolNeuron 已有 RAG 实现，用户可装它补这个能力）
- 状态: ⏳ 待答

### Q4. 项目名 "MiBrain" 是否最终确认
- 候选: MiBrain / HyperBrain / K5Brain / 自定义
- **建议**: MiBrain（与 HyperOS 对应，有辨识度）
- 状态: ⏳ 待答

## 已确认（来自之前的设计讨论）

### ✅ Root 环境
KernelSU + ZygiskNext + LSPosed（用户已确认）

### ✅ 设备
红米 K50 Ultra（骁龙 8+ Gen 1, 8GB RAM, HyperOS）

### ✅ 推理后端
llama.cpp b9844 + llama-server 二进制（不再用 JNI 集成）

### ✅ ASR/TTS 引擎
sherpa-onnx v1.13.3 AAR

### ✅ LSPosed 方案
装现成的 Phantom Mic，不自写 hook

### ✅ APK 开发语言
Kotlin + Jetpack Compose

### ✅ Daemon 选型
Shell + 现成二进制（不用 Go / 不用 Termux）

### ✅ 仓库归属
GitHub 用户名 `qbjsdsb`，仓库名 `mibrain`

### ✅ 借鉴清单
详见 [docs/00_design_overview.md](./00_design_overview.md) §3
