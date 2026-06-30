# 项目骨架说明

> 此目录是项目骨架，**当前为空**，等 Phase 1 启动后填入实际代码。

## 计划结构（Phase 1 完成后）

```
app/                                # MiBrain APK（Android Studio 项目）
├── app/
│   ├── build.gradle.kts            # Gradle 配置
│   ├── src/main/
│   │   ├── AndroidManifest.xml     # 含 RECORD_AUDIO, FOREGROUND_SERVICE 等权限
│   │   ├── java/com/mibrain/
│   │   │   ├── MainActivity.kt     # Compose 入口
│   │   │   ├── service/
│   │   │   ├── engine/
│   │   │   ├── data/
│   │   │   ├── ui/
│   │   │   └── util/
│   │   ├── assets/
│   │   └── jniLibs/arm64-v8a/      # sherpa-onnx 的 .so（仅 arm64-v8a ABI）
│   └── proguard-rules.pro
├── settings.gradle.kts
└── gradle/libs.versions.toml       # 版本目录
```

## Phase 1 计划交付的内容

- [ ] Gradle 项目骨架 + libs.versions.toml
- [ ] MainActivity + Compose UI 最简界面
- [ ] LlamaEngine.kt（HTTP 调用本地 llama-server）
- [ ] 一次性对话生成（非流式，非语音）
- [ ] 模型选择 UI

## 不在 Phase 1 范围

- ❌ 录音、ASR、TTS（Phase 2）
- ❌ 唤醒词（Phase 3）
- ❌ LSPosed 配置（Phase 4）
- ❌ CI/CD（Phase 5）

详见 [docs/00_design_overview.md](../docs/00_design_overview.md) §11 路线图。
