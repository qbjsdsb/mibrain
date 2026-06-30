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
- 默认 Qwen2.5-1.5B（[D1](../DECISIONS.md)）在结构化 function calling 上质量不稳（~~3B 可选但同样不保证~~ **3B 在 8GB 设备必 OOM 已不可用**，[D30](../DECISIONS.md)），纯靠 LLM 决策会乱调工具
- 纯关键词扩展性差，每个新工具要改正则
- 混合方案：高频场景（天气/翻译/新闻）走关键词快速路径，长尾场景走 LLM

> **备注（推理后端）**：Stage 2 不走 HTTP 调用 llama-server（[X7](../DECISIONS.md) 已废弃该路径），而是通过 JNI 调用 LlamaEngine.kt 的 `complete()` 或 `streamComplete()`。虽然 JNI 路径下不走 OpenAI 兼容的 `/v1/chat/completions` HTTP 接口，但 llama.cpp 的 chat template 仍支持 function calling 格式（参考 [ToolNeuron](https://github.com/Siddhesh2377/ToolNeuron) 的 `InferenceService.kt` + `InferenceClient.kt`（位于 `service/inference/` 目录）实现），即把 `tools` schema 拼到 prompt 里、解析模型输出中的 tool call 块。

### 1.1 function calling 落地细节（第六轮 R4 补，解决 5 项遗留）

> 第六轮 R4 排查发现：原 §1 仅给出 Stage 1/2 混合架构图，缺 32-token CoT prompt 模板 / 关键词正则表 / fallback 机制 / 多工具测试方案 / 误触发率量化共 5 项。本节补强，作为 Phase 6 实施的**直接依据**。引用 [D25](../DECISIONS.md)（32-token CoT 准确率 64%）+ [D17](../DECISIONS.md)（混合方案）+ [03 §3.1](./03_architecture_detail.md) `completeWithTools()` 接口。

#### 1.1.1 Stage 1 关键词正则表（MVP 必须命中场景，覆盖 80% 流量）

| 工具 | 关键词正则（Kotlin `Regex`，大小写不敏感） | 命中示例 | 排他规则 |
|---|---|---|---|
| WeatherTool | `(天气|气温|下雨|下雪|会不会冷|几度|温度|weather|forecast)` | "明天天气"、"北京几度"、"会不会下雨" | 命中后不再走 Stage 2 |
| TranslateTool | `(翻译|译成|translate\|how to say.*in)` | "翻译这句话"、"how to say hello in 中文" | 命中后切到 TranslateTool |
| NewsTool | `(新闻|热点|头条|今日要闻|news|headline)` | "今日新闻"、"看下热点" | 命中后切到 NewsTool |
| WebSearchTool | **不进 Stage 1**，仅走 Stage 2 | - | 关键词场景太宽（"查一下"、"帮我搜"无边界），强制走 LLM 决策 |

**正则实现规范**（ToolRouter.kt）：
```kotlin
private val weatherRegex = Regex("(?i)(天气|气温|下雨|下雪|会不会冷|几度|温度|weather|forecast)")
private val translateRegex = Regex("(?i)(翻译|译成|translate|how to say.*in)")
private val newsRegex = Regex("(?i)(新闻|热点|头条|今日要闻|news|headline)")

fun routeByKeyword(userText: String): Tool? {
    return when {
        weatherRegex.containsMatchIn(userText) -> WeatherTool
        translateRegex.containsMatchIn(userText) -> TranslateTool
        newsRegex.containsMatchIn(userText) -> NewsTool
        else -> null  // 落到 Stage 2
    }
}
```

**误触发率量化门槛（[D25](../DECISIONS.md) 关联）**：Stage 1 关键词路由的误触发率（命中关键词但用户本意不是调该工具）必须 < 5%。测试方法：Phase 6 真机跑 200 句日常对话（含 50 句含"天气/翻译/新闻"关键词的干扰项，如"今天天气真不错啊"是闲聊不是问天气），统计错误路由次数。失败处理：收紧正则（如要求"天气"前/后有动词"查/看/告诉"），或在 Stage 1 命中后追加一次"确认意图"的 LLM 判定（消耗 1 次轻量 inference）。

#### 1.1.2 Stage 2 LLM function calling（32-token CoT prompt 模板，[D25](../DECISIONS.md)）

Qwen2.5-1.5B 直出 tool call 准确率仅 41%（BFCL v3 实测），加 32-token CoT 后提升到 64%。**prompt 模板固定为以下结构**（CoT 必须 ≤ 32 token，超过反崩到 25%）：

```
<|im_start|>system
你是 MiBrain 语音助手。可选工具：
- WeatherTool: 查询天气，参数 {location: string, date: string}
- TranslateTool: 翻译文本，参数 {text: string, target_lang: string}
- WebSearchTool: 联网搜索，参数 {query: string}
- NewsTool: 获取新闻，参数 {category: string}

用户说话后，先在 <CoT>...</CoT> 标签内用不超过 32 个 token 简短判断是否需要工具（如"用户问天气，调 WeatherTool"），如果需要工具则在 </CoT> 后输出：
<tool>{"name":"WeatherTool","arguments":{"location":"北京","date":"明天"}}</tool>
如果不需要工具则在 </CoT> 后直接回复用户。<|im_end|>
<|im_start|>user
{user_text}<|im_end|>
<|im_start|>assistant
<CoT>
```

**关键约束（native 推理循环实现，[03 §3.1](./03_architecture_detail.md) `completeWithTools()`）**：

```cpp
// llama_engine_jni.cpp: completeWithTools 推理循环（伪代码）
int cot_token_count = 0;
bool in_cot = true;          // 进入 <CoT> 段
bool cot_truncated = false;  // 是否已强制切到 </CoT>

while (true) {
    llama_token tok = llama_sampler_sample(sampler, ctx, batch.n_tokens - 1);

    // Phase 1：CoT 计数（仅限 <CoT> 段内）
    if (in_cot && !cot_truncated) {
        if (++cot_token_count >= 32) {
            // 强制注入 </CoT> 起始 + <tool> 起始检测
            llama_token close_think = lookup_token(vocab, "</CoT>");
            batch = llama_batch_get_one(&close_think, 1);
            in_cot = false;
            cot_truncated = true;
            continue;
        }
    }

    // Phase 2：CoT 结束后只允许 <tool>...</tool> 或纯文本
    if (cot_truncated) {
        // 用 piece buffer 检测 <tool> 起始
        // - 命中 <tool> 则进入 JSON 解析模式
        // - 其他文本作为普通回答流式回调
    }

    if (llama_vocab_is_eog(vocab, tok)) break;
}
```

- `<CoT>...</CoT>` 是项目自定义 CoT 标记（Qwen2.5-Instruct 词表中无此原生 token，需在 native 层用 `llama_tokenize` 把字符串 `"<CoT>"` / `"</CoT>"` 切为多个普通 token 并在 prompt 拼接时预填 `<CoT>` 起始 token）
- CoT token 数由 native 层计数，**不计入 maxTokens 上限**（避免 32-token CoT 占满用户 maxTokens=100 的额度）
- `</CoT>` 后只允许出现 `<tool>...</tool>` 或纯文本，由 ToolRouter.kt 用正则 `Regex("<tool>(\\{.*?\\})</tool>", RegexOption.DOT_MATCHES_ALL)` 解析 JSON
- JSON 解析失败 → fallback 到 Stage 3（直接回退到离线模式纯文本回答）
- `<tool>` 块外的文本一律视为 LLM 普通回答（不调工具）
- 验收门槛（[D25](../DECISIONS.md)）：Phase 6 真机测 100 句工具调用，BFCL 等价准确率 ≥ 64%（与 [D25](../DECISIONS.md) 基准持平）；若 < 50% 则需重评是否引入 0.5B 意图分类器（[D17](../DECISIONS.md) Q5）

#### 1.1.3 fallback 机制（3 级降级）

```
Stage 1 (关键词正则) → 命中则直接调工具
   ↓ 未命中
Stage 2 (LLM 32-token CoT) → 解析 <tool> JSON 成功则调工具
   ↓ JSON 解析失败 / LLM 输出乱 / 超时 5s
Stage 3 (离线纯文本回退) → LLM 基于已有知识直接回答用户（不调任何工具）
   ↓ LLM 也失败（OOM / crash）
Stage 4 (TTS 朗读固定话术) → "抱歉，工具调用失败，请稍后再试"
```

Stage 3 与 Stage 4 之间的差异：Stage 3 仍调 LLM（消耗推理资源），Stage 4 不调 LLM 直接 TTS。Stage 3 准入条件：LLM 进程健康（LlamaEngine 加载正常 + 非处于 native crash 自愈流程）。

#### 1.1.4 多工具并发测试方案

| 测试场景 | 用户输入 | 期望路由 | 验收门槛 |
|---|---|---|---|
| 单工具命中（Stage 1） | "明天天气怎么样" | WeatherTool | 100% 命中（关键词明确） |
| 单工具命中（Stage 2） | "查一下北京明天会不会下雨" | WeatherTool（含"下雨"应走 Stage 1，但"查一下"是搜索信号需测 Stage 2 兜底） | ≥ 80% 命中 |
| 多工具组合 | "翻译'明天天气'成英文" | TranslateTool 优先（嵌套场景按"外层意图"路由） | ≥ 70% 命中 |
| 模糊意图 | "帮我看看最近的事" | Stage 2 → WebSearchTool 或 NewsTool | ≥ 50% 命中（容忍性测试） |
| 无工具需求 | "你好" | Stage 2 → 纯文本回答 | 100% 不调工具 |
| Stage 2 失败回退 | 故意触发 JSON 解析错（构造畸形 LLM 输出） | Stage 3 离线回答 | 100% 不卡死 |

**测试数据集**：Phase 6 真机跑 100 句覆盖上述 6 类场景，统计路由准确率。整体准确率门槛 ≥ 75%（含 Stage 1+2+3 的总和准确率）。失败处理：扩充 Stage 1 关键词正则 / 调整 32-token CoT prompt 措辞 / 评估是否引入 0.5B 意图分类器（[D17](../DECISIONS.md) Q5 暂不做，但若准确率 < 60% 需重评）。

#### 1.1.5 误触发率量化（Stage 1+2 综合）

| 指标 | 门槛 | 测试方法 | 失败处理 |
|---|---|---|---|
| Stage 1 误触发率 | < 5% | 200 句日常对话（含 50 句含"天气/翻译/新闻"的干扰闲聊） | 收紧正则 |
| Stage 2 误触发率 | < 10% | 同上数据集，跑 Stage 2 兜底分支 | 调 32-token CoT prompt |
| 综合（Stage 1+2）误触发率 | < 8% | 加权平均 | 整体路由策略重评 |
| 工具未触发率（漏触发） | < 15% | 100 句明确需要工具的输入 | 扩充关键词 / 改 CoT prompt |

门槛依据：[D25](../DECISIONS.md) Qwen2.5-1.5B + 32-token CoT 准确率 64% 是 BFCL v3 基准，本项目期望通过 Stage 1 关键词快速路径把整体准确率拉到 75%+（关键词场景 100% 准确 + LLM 兜底 64% 加权）。

**反向引用（第六轮 R4 补强）**：本节 §1.1.2 的 32-token CoT prompt 模板 + native 推理循环实现依赖 [15 §2.2](./15_technical_specs.md)（JNI 跨线程 callback 规范——`completeWithTools` 推理循环的 token callback 须在 native 线程 `AttachCurrentThread`，且 Kotlin callback 须 `NewGlobalRef` 持有）+ [15 §3.2](./15_technical_specs.md)（推理循环伪代码——CoT 计数 `cot_token_count` 与 `</CoT>` 强制注入须在该循环内实现，与 EOG 判断 / KV cache 检查同层）。两者共同构成 §1.1.2 native 落地的底层规范。

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
| Q4 | 默认 Qwen2.5-1.5B（[D1](../DECISIONS.md)）的 function calling 是否够稳？~~3B 可选模式是否需在 Phase 6 真机验证不 OOM？~~ **第三轮重算后 3B 在 8GB 设备必 OOM（[D30](../DECISIONS.md)），Phase 6 不可引导用户切 3B**；1.5B function calling 不达标只能靠 [D17](../DECISIONS.md) 关键词正则优先 + 32-token CoT（[D25](../DECISIONS.md)）兜底，或 Phase 11+ 启用 Adreno GPU 加速后再评估 3B | Phase 6 真机测试 1.5B + 32-token CoT 的 function calling 准确率，不达标则强化关键词路由覆盖更多场景 |
| Q5 | 是否新增 0.5B 小模型做意图分类器？ | 暂不，混合方案够用 |
