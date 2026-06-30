# Phase 6 设计：联网工具调用

> 本文档定义 Phase 6 的"联网开关 + 4 个联网工具"设计。  
> 阶段：设计草案（待 Phase 1-5 完成后实施）  
> 依赖：Phase 2（语音链路）+ Phase 5（发布）完成  
> 与主设计关系：[00_design_overview.md](./00_design_overview.md) §11 路线图的扩展

> **2026-06-30 修订说明**（深度检查后第二轮调整）：
> - 推理后端切回 JNI（[D7](../DECISIONS.md) + [X7](../DECISIONS.md)）：Phase 6 工具调用的 LLM function calling 通过 JNI 调用 LlamaEngine.kt，而非 HTTP 调用 llama-server
> - 默认模型从 3B 改为 1.5B（[D1](../DECISIONS.md)）：影响 function calling 准确率，1.5B 可能不如 3B 稳定，需 Phase 6 真机测试
> - 唤醒词改 sherpa-onnx KWS（[D23](../DECISIONS.md)）：本稿不直接涉及，但如提到唤醒词需更新表述

---

## 0. 设计目标

让 MiBrain 在保持"完全离线"为默认模式的前提下，提供一个**全局联网开关**，打开后能调用 4 个联网工具回答用户问题。

**核心约束**：
- 默认关闭联网，保留"隐私不出手机"卖点
- 联网时只发"用户文本 query + 工具所需的最小参数"，不发设备身份/历史对话/原始音频
- 断网降级：开关打开但无网络时，自动 fallback 到离线模式

---

## 1. 架构

```
ConversationEngine
       ↓ (用户文本)
   ToolRouter
       ↓ check networkEnabled
   NetworkGate (全局开关)
       ↓
   ┌────────────────────────────┐
   │ Stage 1: 关键词正则匹配     │
   │  "天气" → WeatherTool       │
   │  "翻译" → TranslateTool     │
   │  "新闻" → NewsTool          │
   │  (其他) → 落到 Stage 2      │
   └────────────────────────────┘
       ↓ 未命中
   ┌────────────────────────────────────┐
   │ Stage 2: LLM function call (JNI)   │
   │  把 4 个工具的 schema 喂给          │
   │  LlamaEngine.kt 的 complete()      │
   │  或 streamComplete()，附带 tools   │
   │  LLM 决定调哪个或直接回答            │
   └────────────────────────────────────┘
       ↓
   Tool 执行 → 拿到结构化结果
       ↓
   LLM 组织语言 → TTS
```

**Stage 1 + Stage 2 混合方案**的取舍（[D17](../DECISIONS.md)）：
- 默认 Qwen2.5-1.5B（[D1](../DECISIONS.md)）在结构化 function calling 上质量不稳，3B 可选但同样不保证，纯靠 LLM 决策会乱调工具
- 纯关键词扩展性差，每个新工具要改正则
- 混合方案：高频场景（天气/翻译/新闻）走关键词快速路径，长尾场景走 LLM

> **备注（推理后端）**：Stage 2 不走 HTTP 调用 llama-server（[X7](../DECISIONS.md) 已废弃该路径），而是通过 JNI 调用 LlamaEngine.kt 的 `complete()` 或 `streamComplete()`。虽然 JNI 路径下不走 OpenAI 兼容的 `/v1/chat/completions` HTTP 接口，但 llama.cpp 的 chat template 仍支持 function calling 格式（参考 [ToolNeuron](https://github.com/Siddhesh2377/ToolNeuron) 的 LlamaEngine.kt 实现），即把 `tools` schema 拼到 prompt 里、解析模型输出中的 tool call 块。

---

## 2. 4 个工具的 API 选型

| 工具 | 主选 API | 备选 | 需要 key | 默认是否启用 |
|---|---|---|---|---|
| WeatherTool | 和风天气 v7（免费 1000 次/天） | Open-Meteo（无 key 免费） | 是（和风）；否（Open-Meteo） | 默认用 Open-Meteo，配置和风 key 后切换 |
| TranslateTool | LibreTranslate 公共实例 | DeepL Free（需 key） | 否 | 启用 |
| WebSearchTool | Bing China Search API | SearXNG 自建 | 是（Bing） | 未配置 key 时禁用 |
| NewsTool | RSS 源聚合（新浪/36氪/澎湃） | NewsAPI | 否 | 启用 |

**默认配置策略**：尽量用免 key 的，让用户即装即用。Bing 搜索需 key，在 Settings 里留配置入口。

---

## 3. 状态机扩展

新增状态 `TOOL_RUNNING`（与 [03_architecture_detail.md §4](./03_architecture_detail.md) 的状态机保持一致，统一通过 CAS 原子转换 `AtomicReference.compareAndSet`）：

```
IDLE → LISTENING → THINKING
                     ↓
              (LLM 决定调工具?)
                     ↓
              TOOL_RUNNING (联网工具执行中)
                     ↓
              THINKING (LLM 组织语言)
                     ↓
              SPEAKING → COOLDOWN → IDLE
```

**状态转移（CAS 原子转换）**：
- `THINKING → TOOL_RUNNING`：通过 `state.compareAndSet(THINKING, TOOL_RUNNING)` 原子完成，避免并发抢占（如通知到达 Phase 7 与工具调用同时发生）
- `TOOL_RUNNING → THINKING`：工具完成后通过 CAS 转回 THINKING
- 与 [03_architecture_detail.md §4](./03_architecture_detail.md) 实现一致：`transitionTo(expected, new)` 统一封装 `compareAndSet`，禁止直接 `set()` 跳变

**`TOOL_RUNNING` 期间**：
- TTS 播报"正在查询..."安抚用户（避免 3-5 秒静默）
- 唤醒词检测暂停（避免用户重复喊触发混乱）
- 5 秒超时未返回则提示"查询失败"，CAS 回 IDLE

---

## 4. 联网开关 UI

```
Settings → 网络
├── [总开关] 允许联网     ▢
├── [子开关] 允许天气      ▢ (依赖总开关)
├── [子开关] 允许翻译      ▢
├── [子开关] 允许搜索      ▢
├── [子开关] 允许新闻      ▢
├── API Keys
│   ├── 和风天气 key: [______]
│   ├── Bing 搜索 key: [______]
│   └── (LibreTranslate/News 不需要)
└── 隐私
    ├── □ 联网请求记录到日志
    └── □ 离线模式（强制关闭所有联网工具）
```

**默认值**：
- 总开关：关（保留离线卖点）
- 各子开关：开（总开关打开后即可用，除非用户单独关）
- 隐私 → 联网请求日志：关
- 隐私 → 离线模式：关

---

## 5. 隐私边界

**会发送到云端的内容**：
- 用户语音转出的文本 query（如"明天天气"会作为参数发给和风天气 API）
- 工具所需的最小参数（如 GPS 经纬度发给和风天气）

**不会发送**：
- 用户身份（账号、设备 ID）
- 历史对话上下文
- ASR 原始音频
- TTS 输出

**日志**：默认不记录联网请求 body，只在 Settings → 网络 → 隐私里勾选"联网请求记录到日志"才记录（用于排查问题）。

**断网降级**：网络开关打开但实际没网时，自动 fallback 到离线模式，不报错，让 LLM 直接回答（基于模型已有知识）。

---

## 6. 数据流时序（一次天气查询）

```
用户："明天天气"
  ↓ ASR (~1s)
"明天天气"
  ↓ ToolRouter
关键词命中 → WeatherTool
  ↓ NetworkGate check (开关 ON, 子开关 ON)
  ↓ 请求位置 (GPS 缓存, ~0.5s)
  ↓ 调 和风天气 API (~1.5s)
  ↓ 拿到 JSON: {temp: 18, weather: "晴", ...}
  ↓ LLM 组织语言 (~1s)
"明天天气晴，气温 18 度，适合出门"
  ↓ TTS (~0.5s)
播放

总延迟: ~4-5s
```

---

## 7. 位置获取

**方式**：GPS 定位（Android `LocationManager`）

**实现细节**：
- APK 启动时申请 `ACCESS_FINE_LOCATION` 权限
- 用户首次开启天气工具时引导授权
- 位置缓存：每 30 分钟刷新一次，避免每次查询都开 GPS 耗电
- 缓存命中时延迟 <100ms，未缓存时延迟 1-3s（首次 GPS lock）

**降级**：
- 用户拒绝定位权限 → 提示"天气功能需要定位权限，请在系统设置里授予"
- GPS 长时间无信号 → 提示"无法获取位置，请检查 GPS 是否开启"

---

## 8. 工程量估算

| 模块 | 行数估算 | 备注 |
|---|---|---|
| Tool.kt 接口 + ToolRouter + NetworkGate | 200 | 基础设施 |
| 状态机扩展（TOOL_RUNNING 状态） | 100 | |
| WeatherTool | 150 | 含位置缓存 |
| TranslateTool | 150 | |
| WebSearchTool | 300 | 抓网页 + readability |
| NewsTool | 200 | RSS 解析 |
| UI 设置页 | 300 | Compose |
| API key 持久化 | 100 | Room |
| 测试 + 集成 | 200 | |
| **合计** | **~1700 行 Kotlin** | 约 Phase 2 工程量 |

---

## 9. 范围外（明确不做）

- ❌ 自动获取位置权限弹窗（用户首次开天气工具时引导）
- ❌ 离线缓存 API 响应（Keep it simple，每次都查最新）
- ❌ 多个翻译 API fallback 切换（用户自己改 key）
- ❌ 搜索结果分页（一次拿 top 3 摘要就够）
- ❌ 自定义 RSS 源（MVP 用预设 3 个源：新浪/36氪/澎湃）
- ❌ 多模态（OCR、看图）—— 留 Phase 9
- ❌ 插件系统 —— 留 Phase 8

---

## 10. 与主路线图的关系

[README.md](../README.md) 与 [00_design_overview.md §11](./00_design_overview.md) 路线图后续补 Phase 6-9：

| Phase | 内容 | 状态 |
|---|---|---|
| 0 | 设计冻结 | ✅ 当前 |
| 1 | MVP 单链路 | ⏳ 待启动 |
| 2 | 语音链路 | ⏳ |
| 3 | 唤醒词 | ⏳ |
| 4 | 稳定性 | ⏳ |
| 5 | 发布 | ⏳ |
| **6** | **联网工具调用**（本文档） | **设计草案** |
| 7 | 手机控制（通知朗读/应用启动/系统设置/截屏） | 待 brainstorm |
| 8 | 插件系统 + API 暴露 | 待 brainstorm |
| 9 | 多模态（OCR/看图） | 待 brainstorm |

---

## 11. 待 Phase 5 实施前再确认的开放问题

| # | 问题 | 当前默认 |
|---|---|---|
| Q1 | 和风天气 v7 API 是否仍在免费档？ | 默认假设仍可用，Phase 6 实施前复查 |
| Q2 | LibreTranslate 公共实例是否仍稳定？ | 默认假设仍可用，Phase 6 实施前复查 |
| Q3 | Bing China Search API 是否仍可用？ | 默认假设仍可用，否则换 SearXNG |
| Q4 | 默认 Qwen2.5-1.5B（[D1](../DECISIONS.md)）的 function calling 是否够稳？3B 可选模式是否需在 Phase 6 真机验证不 OOM？ | Phase 6 真机测试 1.5B function calling 准确率，不达标则引导用户切 3B |
| Q5 | 是否新增 0.5B 小模型做意图分类器？ | 暂不，混合方案够用 |
