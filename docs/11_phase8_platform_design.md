# Phase 8 设计：平台化能力

> 本文档定义 Phase 8 的 3 个平台化能力设计。  
> 阶段：设计草案（待 Phase 1-7 完成后实施）  
> 依赖：Phase 6（联网工具调用，提供 Tool 接口）+ Phase 7（手机控制工具）完成  
> 与主设计关系：[00_design_overview.md](./00_design_overview.md) §11 路线图的扩展

> **2026-06-30 修订说明**（深度检查后第二轮调整）：
> - **Cap 1 本地 API 暴露重新设计**（[D7](../DECISIONS.md) + [X7](../DECISIONS.md)）：原设计复用 llama-server 的 8080 端口对外暴露，但 llama-server HTTP 路径已废弃，切回 JNI 后需要在 APK 内嵌入一个 HTTP server（如 NanoHTTPD 或 Ktor）作为 LlamaEngine 的对外门面
> - 推理后端切回 JNI（[D7](../DECISIONS.md)）
> - 默认模型从 3B 改为 1.5B（[D1](../DECISIONS.md)）
> - 唤醒词改 sherpa-onnx KWS（[D23](../DECISIONS.md)）

---

## 0. 设计目标

让 MiBrain 从"单一应用"升级为"平台"，让其他应用/设备能调用其能力：

| Cap | 做什么 | 主要受众 |
|---|---|---|
| Cap 1: 本地 API 暴露 | 在 APK 内嵌入 HTTP server 暴露 LlamaEngine 能力，加 UI 配置 + 限流 + token 计数 | 同机开发者、Tasker/MacroDroid 用户 |
| Cap 3: 桌面小组件 | 桌面长按放图标，一键进入对话/听写模式 | 终端用户 |
| Cap 4: MCP 协议支持 | MiBrain 当 MCP server，桌面端 IDE 可调 | 桌面端开发者 |

**明确不做**（用户决策已确认）：Cap 2 插件系统（安全风险大，跳过）。

---

## 1. Cap 1: 本地 API 暴露

### 1.1 现状与修订

> ⚠️ **修订**（[D7](../DECISIONS.md) + [X7](../DECISIONS.md)）：原设计复用 llama-server 的 8080 端口对外暴露，但 llama-server HTTP 路径已废弃，切回 JNI 后需要在 APK 内嵌入一个 HTTP server 作为 LlamaEngine 的对外门面。

切回 JNI（[D7](../DECISIONS.md)）后，LlamaEngine 通过 JNI 在 APK 进程内提供推理能力，不再有 llama-server 常驻 HTTP 端点。要对外暴露 LLM 能力，需在 APK 内嵌入一个轻量 HTTP server，由它接收请求 → 调用 LlamaEngine JNI → 返回结果。

### 1.2 HTTP server 选型（2026-06-30 第三轮修订：E7，NanoHTTPD → Ktor）

> ⚠️ **2026-06-30 第三轮修订**（[E7](./14_feasibility_recheck_and_plan.md) + [D28](../DECISIONS.md)）：原表把 NanoHTTPD 标为"推荐"是误判，第三轮 web 核实发现：
> - NanoHTTPD **last commit ≈ 2 年前**（约 2024 年中），项目实质已停止维护（[open-awesome.com 实测](https://open-awesome.com/projects/nanohttpd)）
> - 仅支持 HTTP 1.1，**不支持 HTTP/2/HTTP/3**
> - 无内置 SSE（Server-Sent Events），需自实现，与 Phase 8 Cap 1 流式输出需求冲突
> - 无内置认证、日志、路由系统
> - WebSocket 是独立模块（非内置）
> - 7.2k stars 但 162 open issues 无人响应
>
> 选型改为 **Ktor**（[D28](../DECISIONS.md)）：体积增量 ~5MB 可接受（APK 已含 libllama.so 30MB + sherpa-onnx 19MB），换来原生 SSE/路由/中间件 + 活跃维护 + 与 Kotlin coroutines 自然集成。

| 选项 | 体积 | 优点 | 缺点 | 第三轮核实结论 |
|---|---|---|---|---|
| ~~NanoHTTPD~~ | ~50KB | 轻量，零额外依赖，Android 友好 | 功能基础，需自实现 SSE | ❌ **last commit 2 年前，实质停止维护**；仅 HTTP 1.1；无内置 SSE；**不推荐** |
| **Ktor（推荐）** | ~5MB | 功能全，原生支持 SSE/路由/中间件，活跃维护 | 体积比 NanoHTTPD 大 | ✅ JetBrains 官方活跃维护；与 Kotlin coroutines 自然集成；Phase 8 流式输出 + Cap 4 MCP JSON-RPC 都受益 |

### 1.3 架构

```
[外部请求] → Ktor server (APK 内嵌, 8080) → LlamaEngine (JNI) → llama.cpp → 返回
```

HTTP server 在 APK 进程内运行，直接调用 LlamaEngine.kt（项目自命名；参考 [ToolNeuron](https://github.com/Siddhesh2377/ToolNeuron) 的 `InferenceService.kt` + `InferenceClient.kt`（位于 `service/inference/` 目录）实践），不跨进程/不跨 SELinux 域。

### 1.4 路径设计

| 路径 | 方法 | 说明 |
|---|---|---|
| `/v1/chat/completions` | POST | OpenAI 兼容，调 `llamaEngine.complete()` 或 `streamComplete()` |
| `/health` | GET | 健康检查，返回 `{"status":"ok"}` |

端口默认 8080，可在 `config.json` 的 `network.api.port` 字段配置。

### 1.5 实现草案（2026-06-30 第三轮修订：NanoHTTPD → Ktor）

```kotlin
// Ktor 实现（替代原 NanoHTTPD 草案，[D28](../DECISIONS.md)）
fun Application.mibrainApiModule(llamaEngine: LlamaEngine) {
    routing {
        post("/v1/chat/completions") {
            // 调 llamaEngine.complete() 或 streamComplete()
            // 流式响应用 respondTextWriter + chunked transfer
            val req = call.receive<ChatRequest>()
            if (req.stream == true) {
                call.respondTextWriter(ContentType.Text.EventStream) {
                    llamaEngine.streamComplete(req.toPrompt(), req.maxTokens, req.temperature) { token ->
                        write("data: $token\n\n")
                        flush()
                    }
                    write("data: [DONE]\n\n")
                }
            } else {
                val result = llamaEngine.complete(req.toPrompt(), req.maxTokens, req.temperature)
                call.respond(ChatResponse(result))
            }
        }
        get("/health") {
            call.respondText("""{"status":"ok"}""", ContentType.Application.Json)
        }
    }
}
```

Ktor 与 llama.android 官方模块 + ToolNeuron 的 Kotlin 全栈一致，无额外语言负担。

### 1.6 Phase 8 增强项

| 项 | 实现 | 默认值 |
|---|---|---|
| 端口配置 | Settings → 平台 → API 端口（写入 `config.json` 的 `network.api.port`） | 8080 |
| 绑定地址 | localhost / LAN 二选一 | localhost |
| 鉴权 | Bearer token（可关闭） | 关 |
| 限流 | 每分钟请求数上限 | 60 |
| token 计数 | 每次请求记录 input/output tokens | 开 |
| 用量统计 | UI 显示今日/本周/总 token 数 | 开 |
| CORS | 是否允许浏览器跨域 | 关 |
| 健康检查端点 | `/health` 端点 | 开 |

### 1.7 LAN 暴露（可选开启）

默认仅 127.0.0.1，避免被同 Wi-Fi 设备未授权调用。用户在 Settings 里手动开启 LAN 暴露时：
- 强制弹出"局域网暴露会让同 Wi-Fi 设备能调你的 MiBrain，建议同时开启 Bearer token"
- 用户配置 token，其他设备调用时需在 Authorization 头带上
- 自带 IP 显示（自动检测本机 IP），方便用户在其他设备配置

### 1.8 鉴权流程

```
其他设备请求:
POST http://192.168.1.100:8080/v1/chat/completions
Authorization: Bearer <token>
Content-Type: application/json

{
  "model": "qwen2.5-1.5b",
  "messages": [{"role": "user", "content": "你好"}]
}
```

**未配置鉴权时**（默认）：仅 localhost 可访问，不需 token。
**配置了鉴权时**：所有请求都需带 Bearer token，否则 401。

### 1.9 token 计数

HTTP server 在响应里附带 OpenAI 协议的 `usage` 字段，每次响应记录：
- input_tokens
- output_tokens
- 模型名
- 调用来源 IP（如果是 LAN 暴露）

UI 展示：
```
Settings → 平台 → 用量统计
├── 今日: 1234 input / 567 output tokens
├── 本周: 8901 / 2345
└── 总计: 23456 / 7890
└── 按来源 IP 分布（仅 LAN 暴露时）
```

---

## 2. Cap 3: 桌面小组件

### 2.1 三个 widget 类型

#### Widget 1: 一键对话

```
┌─────────────┐
│   💬 MiBrain  │
│  对话模式    │
└─────────────┘
```
点击 → 启动 MiBrain → 直接进入对话模式（跳过首页）

#### Widget 2: 一键听写

```
┌─────────────┐
│   📝 听写    │
│  转文字模式  │
└─────────────┘
```
点击 → 启动 MiBrain → 进入听写模式（不调 LLM，只 ASR → 输出到剪贴板）

#### Widget 3: 状态卡（4×2 大尺寸）

```
┌────────────────────────┐
│ MiBrain   ● 在线  CPU 32% │
│ 模型: Qwen2.5-1.5B         │
│ 今日: 12 次对话            │
│ [对话]    [听写]          │
└────────────────────────┘
```
实时显示状态 + 两个按钮。

### 2.2 实现

- AppWidgetProvider + Glance（Jetpack Glance 是 Compose 风格的 widget 框架）
- RemoteViewsService 数据更新（每 30s 刷新一次状态卡）
- 点击通过 PendingIntent 启动 MainActivity + extra 参数路由

### 2.3 工程量

| 模块 | 行数 |
|---|---|
| 一键对话 widget | 100 |
| 一键听写 widget | 100 |
| 状态卡 widget | 150 |
| Glance + 数据更新 | 100 |
| Widget 配置 Activity | 50 |
| **合计** | **~500 行** |

---

## 3. Cap 4: MCP 协议支持

### 3.1 MCP 简介（2026-06-30 第三轮核实：MCP Kotlin SDK 归属修正）

> ⚠️ **E8 第三轮 web 核实补正**（[E8](./14_feasibility_recheck_and_plan.md) + [D28](../DECISIONS.md)）：原表"Anthropic 提出的开放协议"表述准确（MCP 协议规范确实由 Anthropic 主导发布），但**MCP Kotlin SDK 实际由 `modelcontextprotocol` 官方组织维护**（非 Anthropic 仓库直接维护），与 Java/Python/TypeScript SDK 一致：
> - **官方仓库**：[github.com/modelcontextprotocol/kotlin-sdk](https://github.com/modelcontextprotocol/kotlin-sdk)（modelcontextprotocol 官方组织）
> - **数据**：1393★，0.13.0 版本，活跃维护（weekly commits）
> - **Android 兼容性**：第三方已有现成 Android PoC [github.com/kaeawc/android-mcp-sdk](https://github.com/kaeawc/android-mcp-sdk)（Android 端集成参考样板）
> - **协议规范**：MCP 协议规范本身（含 JSON-RPC over stdio/HTTP+SSE）由 Anthropic 文档站 [modelcontextprotocol.io](https://modelcontextprotocol.io) 维护
> - 与 D7 一致性：MCP Kotlin SDK 与 llama.android + ToolNeuron 全栈 Kotlin 一致，无新语言负担

MCP（Model Context Protocol）是 Anthropic 提出的开放协议，让 LLM 客户端（如 Claude Desktop、Cursor、VS Code）能调用外部工具/资源。MiBrain 当 MCP server，桌面端就能调手机上的能力。

### 3.2 暴露的 MCP 工具

| MCP 工具名 | 做什么 | 实际调用 |
|---|---|---|
| `mibrain.chat` | 调本地 LLM 推理 | MiBrain HTTP server (Cap 1, 8080) |
| `mibrain.asr` | 音频 → 文本 | sherpa-onnx ASR |
| `mibrain.tts` | 文本 → 音频 | sherpa-onnx TTS |
| `mibrain.weather` | 查天气（需联网开关） | Phase 6 WeatherTool |
| `mibrain.notification.read` | 读最新通知 | Phase 7 NotificationReaderTool |
| `mibrain.screenshot` | 截屏 + vision 描述 | Phase 7 ScreenCaptureTool |
| `mibrain.app.launch` | 启动 app | Phase 7 AppLauncherTool |
| `mibrain.system.setting` | 切换系统设置 | Phase 7 SystemSettingsTool |

### 3.3 协议实现

MCP 用 JSON-RPC over stdio 或 HTTP+SSE。MiBrain 在手机端跑 HTTP+SSE server，桌面端通过 `mcp.json` 配置连接：

```json
{
  "mcpServers": {
    "mibrain": {
      "url": "http://192.168.1.100:9090",
      "headers": { "Authorization": "Bearer <token>" }
    }
  }
}
```

### 3.4 跨设备协议适配

| 方向 | 实现 |
|---|---|
| MiBrain → 桌面端 | 桌面端发 JSON-RPC 请求，MiBrain 响应 |
| MiBrain → 本机 app | 通过 Phase 8 Cap 1 的 8080 端口（仅 LLM）；MCP 工具走 9090 端口（其他能力） |

### 3.5 安全

- 默认仅 localhost，需手动开 LAN
- 强制 Bearer token 鉴权
- 每个 MCP 工具单独开关（默认全关，用户按需开启）
- `mibrain.notification.read` 默认关闭（最敏感）

### 3.6 工程量

| 模块 | 行数 |
|---|---|
| MCP JSON-RPC 协议实现 | 200 |
| 8 个工具适配器 | 400 |
| SSE 推送（流式 LLM 输出） | 100 |
| UI 配置 + 鉴权 | 100 |
| 测试 + 集成 | 100 |
| **合计** | **~900 行** |

---

## 4. UI 设计

```
Settings → 平台
├── Cap 1: 本地 API 暴露
│   ├── [开关] 启用本地 API   ▢
│   ├── 端口: [8080]
│   ├── 绑定地址: ( ) localhost  ( ) LAN
│   ├── [开关] Bearer token 鉴权  ▢
│   ├── Token: [随机生成]
│   ├── 限流: [60] req/min
│   ├── [开关] 记录 token 计数  ▢
│   └── 用量统计: 今日 1234 / 567 tokens
├── Cap 3: 桌面小组件
│   ├── [按钮] 添加对话 widget 到桌面
│   ├── [按钮] 添加听写 widget 到桌面
│   └── [按钮] 添加状态卡 widget 到桌面
└── Cap 4: MCP 协议
    ├── [开关] 启用 MCP server  ▢
    ├── 端口: [9090]
    ├── 绑定地址: ( ) localhost  ( ) LAN
    ├── Token: [随机生成]
    └── 暴露的 MCP 工具
        ├── □ mibrain.chat
        ├── □ mibrain.asr
        ├── □ mibrain.tts
        ├── □ mibrain.weather
        ├── □ mibrain.notification.read
        ├── □ mibrain.screenshot
        ├── □ mibrain.app.launch
        └── □ mibrain.system.setting
```

---

## 5. 工程量汇总

| Cap | 行数 | 备注 |
|---|---|---|
| Cap 1: 本地 API 暴露 | 300 | UI 配置 + 限流 + token 计数 |
| Cap 3: 桌面小组件 | 500 | 3 个 widget |
| Cap 4: MCP 协议 | 900 | 协议 + 8 个工具适配 |
| **合计** | **~1700 行** | |

---

## 6. 范围外（明确不做）

- ❌ Cap 2: 插件系统（用户决策已跳过，安全风险大）
- ❌ 自定义 widget 主题/样式（用系统默认 Material You）
- ❌ MCP server 的 stdio 模式（只做 HTTP+SSE，简化）
- ❌ API 用量计费（只统计不收费）
- ❌ 多 LLM 模型同时暴露（只暴露当前激活的）

---

## 7. 风险与缓解

| 风险 | 概率 | 影响 | 缓解 |
|---|---|---|---|
| LAN 暴露被同 Wi-Fi 攻击 | 中 | 高 | 强制 Bearer token + 默认 localhost |
| MCP 协议版本演进 | 中 | 中 | 锁版本到 MCP 1.0，Phase 8 实施前复查 |
| Widget 在 MIUI 上被省电策略杀 | 高 | 低 | 系统级电池白名单已配置（Phase 4） |
| HTTP server 限流不生效导致 OOM | 中 | 中 | 单 IP 限流 + 全局 token 上限 |
| MCP 工具调用越权（如不该让外部调 system.setting） | 中 | 高 | 每个工具单独开关 + 默认全关 |

---

## 8. 与 Phase 6/7 的关系

- **Cap 1** 不依赖 Phase 6/7（LlamaEngine (JNI) + 内嵌 HTTP server 是 Phase 1 基础设施）
- **Cap 3** 仅依赖 Phase 1-3（widget 触发的功能是基础对话/听写）
- **Cap 4** 强依赖 Phase 6/7（MCP 工具要复用 Phase 6 的 4 个联网工具 + Phase 7 的 4 个手机控制工具）

如果 Phase 8 与 Phase 6/7 并行开发，Cap 4 可以最后做。

---

## 9. 待实施前再确认的开放问题

| # | 问题 | 当前默认 |
|---|---|---|
| Q1 | MCP 协议版本是否仍是 1.0？ | 假设是，Phase 8 实施前复查 |
| Q2 | Jetpack Glance 是否仍稳定？ | 假设是，Phase 8 实施前复查 |
| Q3 | LAN 暴露默认端口是否会被 HyperOS 防火墙拦？ | 假设不会，实测确认 |
| Q4 | 是否需要给 MCP server 加 rate limit？ | 默认不加（依赖 Cap 1 的限流） |
