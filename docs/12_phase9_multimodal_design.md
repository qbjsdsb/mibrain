# Phase 9 设计：多模态（看图对话）

> 本文档定义 Phase 9 的多模态能力设计。  
> 阶段：设计草案（待 Phase 1-8 完成后实施）  
> 依赖：Phase 6（联网开关，强制开启）+ Phase 5（发布）完成  
> 与主设计关系：[00_design_overview.md](./00_design_overview.md) §11 路线图的扩展

> **2026-06-30 修订说明**（深度检查后第二轮调整）：
> - **S9 修复**：状态机加 COOLDOWN 过渡，IMAGE_CAPTURING 完成后应通过 COOLDOWN 回 IDLE（与 [03_architecture_detail.md §4](./03_architecture_detail.md) 一致）
> - 推理后端切回 JNI（[D7](../DECISIONS.md)）
> - 默认模型从 3B 改为 1.5B（[D1](../DECISIONS.md)）
> - 多模态视觉 API 选 GLM-4V（[D20](../DECISIONS.md)）
> - 模型路径改 DE 加密区（[D21](../DECISIONS.md)）

---

## 0. 设计目标

让 MiBrain 能"看图"——用户拍照后，能与图片内容进行单轮或多轮对话。  
**关键约束**：与"完全离线"定位冲突，所有功能**强制依赖联网开关**（Phase 6）。

---

## 1. 两个能力（实为同一基础设施两种用法）

| 能力 | 描述 | 用法 |
|---|---|---|
| OCR / 单轮看图 | "这是什么药" → 拍照 → 单次描述 | Cap 1 |
| 看图对话 | "拍一张图" → 多轮对话问图 | Cap 3 |

**统一架构**：Cap 3 是 Cap 1 的超集。MVP 实现"看图对话"主功能，OCR 作为简化入口（不进入对话循环，一次性问答）。

---

## 2. 拍照入口

### 2.1 三种拍照方式

| 方式 | 实现 | 默认 |
|---|---|---|
| 调用系统相机 | `Intent(MediaStore.ACTION_IMAGE_CAPTURE)` | ✅ 默认 |
| 直接相机 API | `CameraX` | 备选 |
| 从相册选 | `Intent(Intent.ACTION_PICK)` | ✅ 同时支持 |

### 2.2 触发方式

| 触发 | 流程 |
|---|---|
| 语音触发 | "拍照看这是什么" / "看图对话" → ASR → 路由到 ImageChatTool → 弹相机 |
| UI 按钮 | 设置页 → 多模态 → "拍一张"按钮 |
| 上次拍过的图 | "看上次拍的图" → 加载最近 1 张到对话上下文 |

---

## 3. 流程

### 3.1 OCR / 单轮看图（Cap 1）

```
用户："这是什么药"
  ↓ ASR → "这是什么药"
  ↓ ToolRouter → 关键词"这是什么" → ImageChatTool, mode=single
  ↓ NetworkGate check (开关必须开)
  ↓ state = TOOL_RUNNING (CAS: IDLE → TOOL_RUNNING)
  ↓ TTS："好的，请拍照"
  ↓ state = IMAGE_CAPTURING (CAS: TOOL_RUNNING → IMAGE_CAPTURING)
  ↓ 启动相机 intent
  ↓ 用户拍完照返回
  ↓ state = TOOL_RUNNING (CAS: IMAGE_CAPTURING → TOOL_RUNNING)
  ↓ 图片转 base64
  ↓ 调 vision API: "What is this? {image}"（GLM-4V，[D20](../DECISIONS.md)）
  ↓ state = THINKING (CAS: TOOL_RUNNING → THINKING)
  ↓ LlamaEngine (JNI) 组织语言（[D7](../DECISIONS.md)）
  ↓ state = SPEAKING (CAS: THINKING → SPEAKING，首 token 出)
  ↓ TTS："这是一盒布洛芬缓释胶囊..."
  ↓ 播放完毕 + 500ms
  ↓ state = COOLDOWN (CAS: SPEAKING → COOLDOWN)
  ↓ state = IDLE (CAS: COOLDOWN → IDLE)
```

### 3.2 看图对话（Cap 3）

```
用户："看图对话"
  ↓ ASR → "看图对话"
  ↓ ToolRouter → 关键词"看图对话" → ImageChatTool, mode=multi
  ↓ state = TOOL_RUNNING (CAS: IDLE → TOOL_RUNNING)
  ↓ TTS："好的，请拍照，拍完我们可以聊"
  ↓ state = IMAGE_CAPTURING (CAS: TOOL_RUNNING → IMAGE_CAPTURING)
  ↓ 启动相机 intent
  ↓ 用户拍完照返回
  ↓ state = TOOL_RUNNING (CAS: IMAGE_CAPTURING → TOOL_RUNNING)
  ↓ 图片转 base64，加入对话上下文
  ↓ 进入多轮对话循环:
  │   state = THINKING (CAS: TOOL_RUNNING → THINKING)
  │   调 vision API: "<image> + history + 用户问题"（GLM-4V，[D20](../DECISIONS.md)）
  │   LlamaEngine (JNI) 回答（[D7](../DECISIONS.md)）
  │   state = SPEAKING (CAS: THINKING → SPEAKING)
  │   TTS："图已加载，你想问什么？" / "图里有 3 个人..."
  │   state = COOLDOWN (CAS: SPEAKING → COOLDOWN，播放完毕 + 500ms)
  │   state = LISTENING (CAS: COOLDOWN → LISTENING，多轮接续听)
  │   用户："图里有多少人"
  │   ASR → 文本
  │   继续/退出？
  ↓ 用户说"退出" → 退出多轮
  ↓ state = COOLDOWN → IDLE (CAS 串)
```

### 3.3 多轮上下文管理

| 行为 | 实现 |
|---|---|
| 图片是否每轮都发 | 是（vision API 是 stateless，必须每次发） |
| 上下文长度 | 保留最近 5 轮 user+assistant 文本（不含图） |
| 多张图片 | 支持，每张图独立 base64，按顺序加入 history |
| 上下文超限 | 优先丢弃最早的图，保留最近一张 |

---

## 4. Vision API 选型

> 决策依据：[D20 多模态视觉 API](../DECISIONS.md)

| API | 优势 | 价格 | 默认 |
|---|---|---|---|
| GLM-4V（智谱） | 国内访问稳，中文好，多模态质量不错 | 收费 | ✅ 默认 |
| GPT-4V | 质量最好 | 收费 | 备选 |
| Claude 3.5 Sonnet | 长文好 | 收费 | 备选 |
| Qwen-VL | 国产，部分场景质量好 | 部分免费 | 备选 |

**与 Phase 7 ScreenCaptureTool 共享配置**：用户在 Settings 里配置一次 vision API key 即可，两个功能共用。

> 注：本节仅指**多模态视觉 API**（看图识别）。本地 LLM 对话推理（如组织语言、多轮对话上下文）走 JNI 调用 `LlamaEngine`（[D7](../DECISIONS.md)），不走 HTTP。

---

## 5. 隐私边界

**会发送到云端的内容**：
- 拍照的图片（base64）
- 用户问题的文本
- 多轮对话上下文（最近 5 轮文本）

**不会发送**：
- 用户身份（账号、设备 ID）
- 之前对话历史（仅当前看图对话内的）
- 原始音频

**强制 UI 告知**（首次启用时）：
> "看图对话功能会将你拍摄的照片发送到 [API 名称] 进行识别。照片内容会泄漏到云端，请勿拍摄敏感信息（密码、私密内容）。是否继续？"

---

## 6. UI 设计

```
Settings → 多模态
├── [开关] 启用看图对话     ▢ (依赖 Phase 6 联网开关)
├── Vision API
│   ├── 提供商: [GLM-4V ▼]
│   ├── API Key: [______]
│   └── □ 我已知晓隐私风险
├── 默认模式
│   ├── ( ) 单轮 OCR（拍完即问）
│   └── ( ) 多轮对话（拍完进入对话）
├── 图片质量
│   ├── ( ) 原图（清晰但慢）
│   ├── ( ) 高压缩（推荐，省流量）
│   └── ( ) 中压缩
└── 历史图片
    ├── □ 保留最近 5 张到对话历史
    └── [按钮] 清除所有看图对话历史
```

---

## 7. 状态机扩展

> 与 [03_architecture_detail.md §4](./03_architecture_detail.md) 状态机一致（S9 修复）。
> 新增状态 `IMAGE_CAPTURING`，并在拍照完成后通过 `TOOL_RUNNING → THINKING → SPEAKING → COOLDOWN → IDLE` 完整闭环，**不能直接 `IMAGE_CAPTURING → IDLE`**（除非 60s 超时兜底）。

完整状态机（含 Phase 9 多模态分支）：

```
IDLE --(用户主动触发 ImageChatTool)--> TOOL_RUNNING
TOOL_RUNNING --(需要拍照)--> IMAGE_CAPTURING
IMAGE_CAPTURING --(用户拍完)--> TOOL_RUNNING
IMAGE_CAPTURING --(60s 超时)--> IDLE
TOOL_RUNNING --(工具完成)--> THINKING
THINKING --(首 token 出)--> SPEAKING
SPEAKING --(播放完毕 + 500ms)--> COOLDOWN
COOLDOWN --(立即)--> IDLE
(SPEAKING，多轮模式) --> COOLDOWN --> LISTENING  # 多轮对话接续听
```

状态图：

```
IDLE → LISTENING → THINKING
                     ↓
              (路由到 ImageChatTool?)
                     ↓
              TOOL_RUNNING (TTS 提示 + 准备拍照)
                     ↓ (需要拍照)
              IMAGE_CAPTURING (相机打开，等待用户拍照)
                     ↓ 用户拍完返回
              TOOL_RUNNING (调 vision API)
                     ↓
              THINKING (LlamaEngine (JNI) 组织语言)
                     ↓ (首 token 出)
              SPEAKING → COOLDOWN (播放完毕 + 500ms)
                     ↓
                 IDLE (单轮模式)
                     ↓ (多轮模式)
              LISTENING (等下一轮问题)
```

### CAS 原子转换（解决 M4 并发抢占）

所有状态转换通过 `AtomicReference.compareAndSet`，与 [03_architecture_detail.md §4](./03_architecture_detail.md) 一致：

```kotlin
private val state = AtomicReference(ConversationState.IDLE)

fun transitionTo(expected: ConversationState, new: ConversationState): Boolean {
    return state.compareAndSet(expected, new)
}
```

拍照路径示例：

```kotlin
// 用户主动触发 ImageChatTool（语音或 UI 按钮）
fun onImageChatTriggered() {
    if (transitionTo(ConversationState.IDLE, ConversationState.TOOL_RUNNING)) {
        // 成功转 TOOL_RUNNING，进入拍照流程
        tts.speak("好的，请拍照")
        transitionTo(ConversationState.TOOL_RUNNING, ConversationState.IMAGE_CAPTURING)
        launchCameraIntent()
    }
}

// 用户拍完照返回
fun onImageCaptured(imagePath: String) {
    if (transitionTo(ConversationState.IMAGE_CAPTURING, ConversationState.TOOL_RUNNING)) {
        // 成功转 TOOL_RUNNING，调 vision API
        val visionResult = glm4vClient.recognize(imagePath.toBase64())
        transitionTo(ConversationState.TOOL_RUNNING, ConversationState.THINKING)
        llamaEngine.streamComplete(visionResult, ...) // LlamaEngine (JNI)
        // 首 token 出 → SPEAKING → COOLDOWN → IDLE（由推理/TTS 回调推进）
    }
}

// 60s 拍照超时兜底
fun onCaptureTimeout() {
    // 强制回 IDLE，丢弃本次拍照
    if (transitionTo(ConversationState.IMAGE_CAPTURING, ConversationState.IDLE)) {
        tts.speak("超时未拍照，已取消")
    }
}
```

**`IMAGE_CAPTURING` 期间**：
- TTS 暂停
- 唤醒词检测暂停（sherpa-onnx KWS，[D23](../DECISIONS.md)）
- 用户可点击"取消"返回 IDLE（CAS：`IMAGE_CAPTURING → IDLE`）
- 60 秒未拍照超时 → CAS 强制回 IDLE → TTS 提示"超时未拍照，已取消"
- **不能直接 `IMAGE_CAPTURING → SPEAKING`**：拍照完成必须先经 `TOOL_RUNNING → THINKING → SPEAKING → COOLDOWN → IDLE` 完整闭环

---

## 8. 工程量估算

| 模块 | 行数估算 | 备注 |
|---|---|---|
| ImageChatTool (单轮 + 多轮) | 400 | |
| 相机 intent + 图片处理 | 150 | base64 + 压缩 |
| 多轮上下文管理 | 100 | |
| UI 设置页 | 200 | |
| 状态机扩展（IMAGE_CAPTURING） | 100 | |
| 测试 + 集成 | 200 | |
| **合计** | **~1150 行** | |

---

## 9. 范围外（明确不做）

- ❌ 视频多模态（不做，成本太高）
- ❌ 图片保存到相册（拍完即用即弃，除非用户主动保存）
- ❌ 多 vision API fallback（用户自己改 key）
- ❌ 离线 OCR（如 PaddleOCR）—— 工程量大，与"完全离线"卖点冲突，留观察
- ❌ 图片编辑（裁剪、旋转）—— 让用户用系统相册先编辑再选

---

## 10. 与 Phase 6/7/8 的关系

| 关系 | 说明 |
|---|---|
| 强依赖 Phase 6 联网开关 | 看图必须联网，开关关时禁用 |
| 共享 Phase 7 ScreenCaptureTool 的 vision API 配置 | 同一个 API key |
| 共享 Phase 7 ScreenCaptureTool 的鉴权与隐私 UI | 复用 |
| 不依赖 Phase 8 | 独立功能 |

---

## 11. 风险与缓解

| 风险 | 概率 | 影响 | 缓解 |
|---|---|---|---|
| 拍照泄漏隐私 | 中 | 高 | 强制 UI 告知 + 默认关闭 + TTS 提示"正在发送图片到云端" |
| vision API 响应慢（5-10s） | 高 | 中 | TTS 播报"正在识别..."安抚；超时 30s 提示失败 |
| 图片 base64 太大导致请求超时 | 中 | 中 | 默认高压缩；提供"原图"选项让用户选择 |
| 多轮上下文累积导致 token 超限 | 中 | 中 | 上下文超限自动丢弃最早图，保留最近 1 张 |
| 相机 intent 在 HyperOS 上行为异常 | 低 | 中 | 备选 CameraX 直接调用 API |

---

## 12. 待实施前再确认的开放问题

| # | 问题 | 当前默认 |
|---|---|---|
| Q1 | GLM-4V API 是否仍可用且价格可接受？ | 假设有，Phase 9 实施前复查 |
| Q2 | 是否需要离线 OCR fallback（如 PaddleOCR）？ | 默认不做，留观察 |
| Q3 | 多轮对话的最大轮数限制？ | 默认无限制，但上下文超 5 轮后丢弃最早图 |
| Q4 | 是否支持拍照前预览 + 重拍？ | 默认支持，系统相机自带 |
