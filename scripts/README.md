# 辅助脚本

> 此目录是辅助脚本，**当前为空**，Phase 1 后填入实际脚本。
>
> **2026-06-30 修订（[D7](../DECISIONS.md) 切回 JNI + 第四轮 [F3](../docs/14_feasibility_recheck_and_plan.md) 版本订正）**：
> - 推理后端从 llama-server HTTP 切回 JNI（[D7](../DECISIONS.md)、[X7](../DECISIONS.md)），`download_binaries.sh` 已废弃，改为 `build_llama_jni.sh`（自编译 libllama.so + libggml.so）
> - llama.cpp 锁定版本从 b9844 订正为 **b9830**（b9844 在 2026-06-30 尚未发布，b9830 为 2026-06-28 最新 release）
> - 默认模型从 3B 改为 1.5B Q4_K_M（[D1](../DECISIONS.md)），3B 在 8GB 设备必 OOM（[03 §6](../docs/03_architecture_detail.md) 第三轮重算）

## 计划脚本

| 脚本 | 用途 | 阶段 |
|---|---|---|
| `build_llama_jni.sh` | 自编译 llama.cpp b9830 → libllama.so + libggml.so（参考 [llama.android](https://github.com/ggml-org/llama.cpp/tree/master/examples/llama.android) 官方模块） | Phase 1 |
| `download_sherpa_aar.sh` | 下载 sherpa-onnx v1.13.3 AAR（arm64-v8a split）到 app/libs/ | Phase 1 |
| `download_models.sh` | 一键下载 Qwen2.5-1.5B Q4_K_M（默认）+ ASR/TTS/VAD/KWS 模型 | Phase 2 |
| `verify_assets.sh` | 校验所有下载文件的 SHA256 | Phase 1 |
| `build_ksu_zip.sh` | 打包 KSU 模块 zip | Phase 1 |
| `ci/build_apk.sh` | CI 中构建 APK（含 JNI 编译） | Phase 5 |
| `ci/verify_design.sh` | CI 中验证设计文档完整性 | Phase 5 |

> **废弃脚本**：
> - ~~`download_binaries.sh`~~（[X7](../DECISIONS.md) 废弃，llama-server HTTP 路径已切回 JNI）
