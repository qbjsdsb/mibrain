# 详细架构与时序图

> 本文档补充 `00_design_overview.md` 中未写清的运行时细节：  
> 二进制启动方式、APK ↔ server 通信协议、回环防护、健康检查、启动顺序。

---

## 1. 进程拓扑（运行时）

```
┌─────────────────────────────────────────────────────────────────────┐
│ Android 系统进程（init 起）                                            │
│                                                                      │
│  boot-completed.sh 触发:                                              │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ llama-server 二进制（root 子进程）                                │ │
│  │   命令行: llama-server \                                          │ │
│  │            --model /sdcard/MiBrain/models/qwen2.5-3b-q4_k_m.gguf │ │
│  │            --host 127.0.0.1 \                                     │ │
│  │            --port 8080 \                                          │ │
│  │            --n-gpu-layers 0 \                                     │ │
│  │            --ctx-size 2048 \                                      │ │
│  │            --threads 4 \                                          │ │
│  │            --cont-batching                                         │ │
│  │   工作目录: /data/adb/mibrain/                                    │ │
│  │   日志: /data/adb/mibrain/llama-server.log                       │ │
│  │   PID 文件: /data/adb/mibrain/llama-server.pid                   │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                          ↑ HTTP 127.0.0.1:8080                       │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ MiBrain APK (uid=10xxx, 非 root)                                │ │
│  │   - ForegroundService (microphone type)                          │ │
│  │   - AudioRecord 持续录音                                         │ │
│  │   - 调用本地 HTTP API                                            │ │
│  │   - sherpa-onnx ASR/TTS in-process                               │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### 关键端口与文件

| 资源 | 路径/端口 | 说明 |
|---|---|---|
| llama-server 监听 | `127.0.0.1:8080` | 只绑 loopback，不暴露外网 |
| llama-server 日志 | `/data/adb/mibrain/llama-server.log` | 滚动，单文件 1MB，保留 3 份 |
| llama-server PID | `/data/adb/mibrain/llama-server.pid` | watchdog 用 |
| 模型目录 | `/sdcard/MiBrain/models/` | 用户用 SAF 选择的目录（兼容） |
| 配置文件 | `/sdcard/MiBrain/config.json` | 用户可改的设置 |

---

## 2. service.sh 启动流程（伪代码）

```bash
#!/system/bin/sh
MODDIR=/data/adb/modules/mibrain

# 1. 等待 /sdcard 挂载（最多等 60 秒）
for i in $(seq 1 60); do
    [ -d /sdcard/MiBrain ] && break
    sleep 1
done

# 2. 检查模型是否就位
MODEL=/sdcard/MiBrain/models/qwen2.5-3b-q4_k_m.gguf
if [ ! -f "$MODEL" ]; then
    echo "[MiBrain] 模型未就位，daemon 不启动" > $MODDIR/install.log
    exit 0
fi

# 3. 启动 llama-server（后台 nohup）
nohup $MODDIR/libs/llama-server \
    --model "$MODEL" \
    --host 127.0.0.1 \
    --port 8080 \
    --n-gpu-layers 0 \
    --ctx-size 2048 \
    --threads 4 \
    --cont-batching \
    > /data/adb/mibrain/llama-server.log 2>&1 &

echo $! > /data/adb/mibrain/llama-server.pid

# 4. 等待 server 起来（最多 30 秒）
for i in $(seq 1 30); do
    if curl -s http://127.0.0.1:8080/health > /dev/null 2>&1; then
        echo "[MiBrain] llama-server ready" >> $MODDIR/install.log
        break
    fi
    sleep 1
done

# 5. 启动 watchdog（每 5 分钟检查一次）
while true; do
    if ! pgrep -f "llama-server" > /dev/null 2>&1; then
        # 重启 llama-server
        nohup $MODDIR/libs/llama-server ... &
    fi
    sleep 300
done &
```

---

## 3. APK ↔ llama-server 通信协议

### 3.1 健康检查

APK 启动时 / ForegroundService 重连时：

```
GET /health
期望: 200 OK, body = {"status": "ok"}
```

如果 5 秒内没响应或返回非 200，APK 显示"服务未就绪，请稍候..."并每 2 秒重试。

### 3.2 一次性对话（Phase 1 MVP）

```
POST /v1/chat/completions
Content-Type: application/json

{
  "model": "qwen2.5-3b-instruct",
  "messages": [
    {"role": "system", "content": "你是 MiBrain，一个本地运行的语音助手。简洁回答。"},
    {"role": "user", "content": "你好"}
  ],
  "max_tokens": 200,
  "temperature": 0.7,
  "stream": false
}

响应: 200 OK
{
  "choices": [
    {"message": {"role": "assistant", "content": "你好！我是 MiBrain..."}}
  ]
}
```

### 3.3 流式对话（Phase 2 起）

```
POST /v1/chat/completions
Content-Type: application/json

{
  ...,
  "stream": true
}

响应: text/event-stream
data: {"choices":[{"delta":{"content":"你好"}}]}
data: {"choices":[{"delta":{"content":"！"}}]}
data: [DONE]
```

APK 按句号/问号/感叹号/换行切句，每完整一句立即送给 sherpa-onnx TTS，不等流结束。

### 3.4 keep-alive 卸载策略

llama-server 自带 `-mlock` 和模型常驻。但我们想 5 分钟空闲后卸载，让 8GB 内存喘口气。

**方案**：通过 llama-server 的 `/slots` 接口查最近活动时间，5 分钟无活动则 kill 进程，下次 APK 调用前由 watchdog 重启。

```
GET /slots
响应: [{"id":0, "last_used_time": 1782818321, ...}]
```

---

## 4. 回环防护协议（Phase 2）

### 问题
TTS 播放声音 → 扬声器 → 麦克风拾起 → ASR 识别 → 当作新命令处理。

### 解决：状态机 + 互斥锁

```
状态枚举:
  IDLE              等待唤醒
  LISTENING         正在录音 ASR
  THINKING          LLM 推理中
  SPEAKING          TTS 播放中
  COOLDOWN          播放完 500ms 冷却（防回声尾巴）

状态转移:
  IDLE --(唤醒词检出)--> LISTENING
  LISTENING --(VAD 静音 1s)--> THINKING
  THINKING --(首 token 出)--> SPEAKING
  SPEAKING --(播放完毕 + 500ms)--> COOLDOWN
  COOLDOWN --(立即)--> IDLE

只在 IDLE 状态下激活唤醒词检测
只在 LISTENING 状态下喂 ASR 数据
SPEAKING + COOLDOWN 期间丢弃所有麦克风数据
```

### 协议层实现

```
APK 内部用一个 AtomicReference<ConversationState>
sherpa-onnx 唤醒检测和 ASR 都先 check state，不是对应状态就丢弃
```

### 进阶：AudioFocus 协作（可选）

TTS 播放前请求 AudioFocus，让其他 App 的媒体播放暂停。播放完释放。这能让回环更小。

---

## 5. 启动顺序与失败处理

### 时序

```
T=0     开机
T=10s   KSU 模块 service.sh 执行
T=12s   llama-server 启动
T=15s   llama-server 加载模型（2GB，~3-5s）
T=20s   llama-server ready，监听 8080
T=30s   系统启动完成，APK 可被启动
T=60s   用户首次打开 APK（或 ForegroundService 自启动）
T=62s   APK 调 GET /health 检查
T=62s   200 OK → 进入 IDLE，激活唤醒词检测
```

### 失败处理

| 故障 | 检测 | 处理 |
|---|---|---|
| service.sh 启动失败 | install.log 有错误 | 显示"模块未就绪，请重启" |
| llama-server 启动失败 | /health 30s 内无响应 | watchdog 重启，3 次失败放弃 |
| 模型加载失败 | llama-server 日志报 OOM | 通知 APK 显示"内存不足" |
| 模型文件不存在 | service.sh 第 2 步退出 | APK 引导用户下载模型 |
| APK 自身被 MIUI 杀 | ForegroundService 死 | 通知 + 自启动白名单引导 |
| Phantom Mic 未启用 | APK 启动时检测 LSPosed scope | 显示"请启用 Phantom Mic 范围" |

---

## 6. 内存预算（重新核算）

8GB 总内存：

| 组件 | 常驻内存 | 峰值内存 | 备注 |
|---|---|---|---|
| HyperOS + 系统 | 5.0GB | 5.5GB | 不可压缩 |
| LSPosed runtime | 100MB | 100MB | |
| Phantom Mic (LSPosed) | 30MB | 30MB | |
| MiBrain APK | 200MB | 250MB | 含 sherpa-onnx |
| llama-server 二进制 | 50MB | 50MB | |
| llama.cpp 加载 Qwen2.5-3B Q4 | 1.8GB | 2.2GB | 含 KV cache |
| sherpa-onnx VAD 常驻 | 50MB | 50MB | |
| sherpa-onnx ASR 推理 | - | 200MB | 仅 LISTENING 期间 |
| sherpa-onnx TTS 推理 | - | 150MB | 仅 SPEAKING 期间 |
| **常驻合计** | **7.23GB** | - | |
| **峰值合计** | - | **8.43GB** | 串行执行时 7.93GB |

**结论**：必须**串行执行 ASR → LLM → TTS**，不能并发。3 阶段峰值约 7.93GB，可接受。

如果系统压力上来，APK 监听 `onTrimMemory(TRIM_MEMORY_RUNNING_CRITICAL)` 主动调用 `/unload` 卸载模型，下次唤醒时重新加载（接受 3-5s 加载延迟）。

---

## 7. 关键时序图（一次完整对话）

```
用户              APK               llama-server        sherpa-onnx        AudioTrack
 │                  │                    │                  │                 │
 │ "hey jarvis"     │                    │                  │                 │
 ├─────────────────►│                    │                  │                 │
 │                  │ 唤醒检出 80ms      │                  │                 │
 │                  │ state=LISTENING    │                  │                 │
 │                  │ ─────────────────────────────────────►│ ASR             │
 │ "明天天气"       │                    │                  │                 │
 ├─────────────────►│                    │                  │                 │
 │                  │ ◄─────────────────────────────────────┤ "明天天气"      │
 │                  │ state=THINKING     │                  │                 │
 │                  │ ──POST /chat──────►│                  │                 │
 │                  │                    │ llama_decode     │                 │
 │                  │ ◄──SSE token───────┤ "明天"          │                 │
 │                  │ ◄──SSE token───────┤ "明天晴"        │                 │
 │                  │ 切句检测"。"        │                  │                 │
 │                  │ state=SPEAKING     │                  │                 │
 │                  │ ─────────────────────────────────────►│ TTS             │
 │                  │ ◄─────────────────────────────────────┤ audio chunk     │
 │                  │ ────────────────────────────────────────────────────────►│ 播放
 │ 听到"明天晴"      │                    │                  │                 │
 │                  │ 播放完毕            │                  │                 │
 │                  │ state=COOLDOWN (500ms) │              │                 │
 │                  │ state=IDLE          │                  │                 │
 │                  │ 重新激活唤醒检测     │                  │                 │
```

总延迟：
- 唤醒检出: 80ms
- ASR 转写: 1.5-3s（5s 录音）
- LLM 首 token: ~1s
- 等首句完成（按 ~10 token）: ~2s
- TTS 合成首句: 0.3s
- **首句总延迟: ~5-7s**

---

## 8. 协议错误码

APK ↔ llama-server 之间的错误码统一：

| HTTP Code | 含义 | APK 处理 |
|---|---|---|
| 200 | 正常 | 解析响应 |
| 400 | 请求格式错 | 显示"内部错误"，记日志 |
| 500 | llama-server 内部错 | 重试一次，仍失败提示用户重启 |
| 503 | 模型未加载 | 显示"正在加载模型..." |
| timeout | llama-server 挂了 | 显示"服务无响应"，watchdog 会自动重启 |

---

## 9. 配置文件 schema

`/sdcard/MiBrain/config.json`:

```json
{
  "model": {
    "path": "/sdcard/MiBrain/models/qwen2.5-3b-q4_k_m.gguf",
    "ctx_size": 2048,
    "threads": 4,
    "keep_alive_minutes": 5
  },
  "asr": {
    "model_dir": "/sdcard/MiBrain/models/paraformer-zh",
    "vad_threshold": 0.5,
    "silence_duration_ms": 1000
  },
  "tts": {
    "model_dir": "/sdcard/MiBrain/models/vits-aishell3",
    "speaker_id": 0,
    "speed": 1.0
  },
  "wakeword": {
    "model": "/sdcard/MiBrain/models/hey_jarvis.onnx",
    "threshold": 0.5,
    "cooldown_ms": 5000
  },
  "system_prompt": "你是 MiBrain，一个本地运行的语音助手。简洁回答。",
  "api": {
    "host": "127.0.0.1",
    "port": 8080
  }
}
```

APK 启动时读这个文件，UI 提供修改界面。

---

## 10. 待补充事项

以下细节在 Phase 1-2 工程实现时再补：

- [ ] APK 内的 LlamaEngine.kt 接口签名
- [ ] sherpa-onnx Kotlin API 调用样例
- [ ] ForegroundService 通知的具体文案
- [ ] 错误重试的具体退避策略
- [ ] Compose UI 各屏幕的 wireframe
