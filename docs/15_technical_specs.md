# MiBrain 技术规范补强（Phase 1 产出前必读）

> 本文档补强 [03_architecture_detail.md](./03_architecture_detail.md) 等设计文档中未充分细化的技术细节，是 Phase 1 产出代码时的**直接依据**。
>
> **2026-06-30 创建**：第五轮深度技术严谨性检查发现 15 项技术细节缺失（5 项 ❌ 阻塞 + 6 项 ⚠️ 强烈建议 + 4 项 🟡 建议），本文档集中补强 ❌ + ⚠️ 共 11 项；🟡 4 项留待对应 Phase 实施时补。
>
> 与主设计关系：[00_design_overview.md](./00_design_overview.md) §11 路线图的技术细化，与 [03_architecture_detail.md](./03_architecture_detail.md) 平级引用。

---

## 1. llama.cpp b9830 编译规范（补强 ❌1）

### 1.1 完整 CMake 选项清单

```bash
# scripts/build_llama_jni.sh 完整命令（CPU-only 版，Phase 0-5 不做 GPU 加速）
NDK_ROOT="${ANDROID_NDK_ROOT:-$ANDROID_NDK_HOME}"
LLAMA_VERSION="b9830"
LLAMA_SRC="$PWD/third_party/llama.cpp"
JNILIBS_DIR="$PWD/app/src/main/jniLibs/arm64-v8a"

# 1. 拉源码（轻量 clone，无历史）
git clone --depth 1 --branch "$LLAMA_VERSION" \
    https://github.com/ggml-org/llama.cpp.git "$LLAMA_SRC"

# 2. CMake configure
cmake -S "$LLAMA_SRC" -B "$LLAMA_SRC/build" \
    -DCMAKE_TOOLCHAIN_FILE="$NDK_ROOT/build/cmake/android.toolchain.cmake" \
    -DANDROID_ABI=arm64-v8a \
    -DANDROID_PLATFORM=android-26 \
    -DANDROID_STL=c++_shared \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_SHARED_LIBS=ON \
    -DGGML_NATIVE=OFF \
    -DLLAMA_BUILD_TESTS=OFF \
    -DLLAMA_BUILD_TOOLS=OFF \
    -DLLAMA_BUILD_EXAMPLES=OFF \
    -DLLAMA_BUILD_SERVER=OFF \
    -DLLAMA_BUILD_APP=OFF

# 3. 编译（只编 llama + ggml 两个 target，避免多余产物）
cmake --build "$LLAMA_SRC/build" --target llama ggml -- -j$(nproc)

# 4. 复制 .so 到 jniLibs
mkdir -p "$JNILIBS_DIR"
cp "$LLAMA_SRC/build/bin/libllama.so" "$JNILIBS_DIR/"
cp "$LLAMA_SRC/build/bin/libggml.so"  "$JNILIBS_DIR/"

# 5. 同时复制 Android NDK 的 libc++_shared.so（c++_shared STL 必备）
cp "$NDK_ROOT/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/lib/aarch64-linux-android/libc++_shared.so" \
   "$JNILIBS_DIR/"
```

### 1.2 选项逐项说明

| 选项 | 值 | 必要性 | 说明 |
|---|---|---|---|
| `CMAKE_TOOLCHAIN_FILE` | `$NDK/build/cmake/android.toolchain.cmake` | 必备 | Android NDK 官方 toolchain |
| `ANDROID_ABI` | `arm64-v8a` | 必备 | K50U 是 arm64，只打一个 ABI 省 2/3 体积 |
| `ANDROID_PLATFORM` | `android-26` | 必备 | Direct Boot 要 API 24+，sherpa-onnx 最低 26（[D5](../DECISIONS.md)）；align [00 §0](./00_design_overview.md) minSdk |
| `ANDROID_STL` | `c++_shared` | 必备 | llama.cpp + sherpa-onnx 共用一份 `libc++_shared.so`，避免静态链接膨胀 |
| `CMAKE_BUILD_TYPE` | `Release` | 必备 | 启用 -O3，禁用 assert |
| `BUILD_SHARED_LIBS` | `ON` | 必备 | 产 `.so` 而非 `.a`，APK jniLibs 必须 `.so` |
| `GGML_NATIVE` | `OFF` | 必备 | 禁用 host-only 优化（避免 x86 runner 编出 x86 指令） |
| `LLAMA_BUILD_TESTS` | `OFF` | 必备 | 避免编译 test 二进制（默认 ON） |
| `LLAMA_BUILD_TOOLS` | `OFF` | 必备 | 避免编译 llama-cli/llama-bench 等工具（默认 ON） |
| `LLAMA_BUILD_EXAMPLES` | `OFF` | 必备 | 避免编译 examples/ 下所有子目录（默认 ON） |
| `LLAMA_BUILD_SERVER` | `OFF` | 必备 | 避免编译 llama-server（[X7](../DECISIONS.md) 已废弃此路径，但 CMake 默认仍 ON） |
| `LLAMA_BUILD_APP` | `OFF` | 必备 | 避免编译 iOS app target（默认 ON） |

### 1.3 .so 输出路径

**实测确认**（llama.cpp b9830 `CMakeLists.txt` 第 13 行）：
```cmake
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
```
→ 编译产物落在 `build/bin/libllama.so` + `build/bin/libggml.so`，**不是** `build/` 根目录。CI 脚本与本地脚本 copy 时必须用 `build/bin/`。

### 1.4 jniLibs 最终结构

```
app/src/main/jniLibs/arm64-v8a/
├── libllama.so              (~20MB, llama.cpp 推理核心)
├── libggml.so               (~10MB, ggml 张量库)
├── libc++_shared.so         (~1MB, C++ 共享 STL，避免重复静态链接)
├── libsherpa-onnx-jni.so    (3.7MB, sherpa-onnx JNI 桥, 来自 AAR)
└── libonnxruntime.so        (15MB, onnxruntime 推理引擎, 来自 AAR)
```
合计约 50MB（与 [03 §1](./03_architecture_detail.md) 体积估算一致）。

### 1.5 Phase 11+ GPU 加速时新增选项（未来，仅记录）

启用 Adreno OpenCL 加速时追加（需 Snapdragon 工具链 Docker 镜像，**Phase 0-5 不做**）：
```bash
-DGGML_OPENCL=ON
-DGGML_HEXAGON=ON
# 产物新增：libggml-opencl.so + libggml-hexagon.so
# 注意：与 sherpa-onnx 的 onnxruntime OpenCL backend 可能冲突，需实测
```

---

## 2. JNI 跨线程 callback 实现规范（补强 ❌2）

### 2.1 问题根因

`JNIEnv*` 是 **thread-local**，不能跨线程传递。LlamaEngine 的 `streamComplete()` 在 native 推理线程触发 token callback，但 `JNIEnv*` 是 Kotlin 主线程的 → 直接保存传递会导致：
- 崩溃（最难复现的 bug）
- 内存泄漏（local ref 不释放）
- Kotlin callback 对象被 GC 回收（native 仍持有悬空指针）

### 2.2 规范实现

```cpp
// llama_engine_jni.cpp

#include <jni.h>
#include <pthread.h>
#include "llama.h"

static JavaVM* g_jvm = nullptr;          // 全局 JavaVM（进程唯一，线程无关）
static jobject g_callback_ref = nullptr; // Kotlin callback 的全局引用

// JNI_OnLoad：JVM 启动时调用一次，保存 JavaVM
JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved) {
    g_jvm = vm;
    JNIEnv* env;
    if (vm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6) != JNI_OK) {
        return JNI_ERR;
    }
    return JNI_VERSION_1_6;
}

// native 推理线程的 callback 入口（由 llama.cpp 调用）
static void on_token_callback(const char* token_str, void* user_data) {
    JNIEnv* env;
    bool attached = false;

    // 1. 在 native 线程重新拿 JNIEnv*
    if (g_jvm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6) == JNI_EDETACHED) {
        if (g_jvm->AttachCurrentThread(&env, nullptr) == JNI_OK) {
            attached = true;
        } else {
            return; // attach 失败，丢弃此 token（不可恢复）
        }
    }

    // 2. 调 Kotlin callback（g_callback_ref 是 NewGlobalRef 持有的，不会被 GC）
    jclass cb_class = env->GetObjectClass(g_callback_ref);
    jmethodID invoke_id = env->GetMethodID(cb_class, "invoke", "(Ljava/lang/String;)V");
    jstring j_token = env->NewStringUTF(token_str);
    env->CallVoidMethod(g_callback_ref, invoke_id, j_token);

    // 3. 释放 local ref（避免栈溢出）
    env->DeleteLocalRef(j_token);
    env->DeleteLocalRef(cb_class);

    // 4. 推理结束 detach（避免线程泄露）
    if (attached) {
        g_jvm->DetachCurrentThread();
    }
}

// Kotlin 调 streamComplete 时，把 callback 转为全局引用
JNIEXPORT jstring JNICALL Java_com_mibrain_engine_LlamaEngine_streamComplete(
    JNIEnv* env, jobject thiz, jstring j_prompt, jint max_tokens, jfloat temp,
    jobject callback) {

    // 保存 callback 为全局引用（避免 GC 回收）
    g_callback_ref = env->NewGlobalRef(callback);

    const char* prompt = env->GetStringUTFChars(j_prompt, nullptr);
    // ... 调 llama.cpp 推理，每 token 调 on_token_callback(token_str, nullptr)
    env->ReleaseStringUTFChars(j_prompt, prompt);

    // 推理结束后释放全局引用
    env->DeleteGlobalRef(g_callback_ref);
    g_callback_ref = nullptr;

    return env->NewStringUTF(full_result.c_str());
}
```

### 2.3 关键约束（必须遵守）

| 约束 | 原因 | 违反后果 |
|---|---|---|
| `JavaVM*` 必须在 `JNI_OnLoad` 保存 | `JNIEnv*` 线程局部，跨线程需用 `JavaVM->GetEnv` | 崩溃 |
| native 推理线程必须 `AttachCurrentThread` | 推理线程非 JVM 创建，无 `JNIEnv*` | `GetEnv` 返回 `JNI_EDETACHED` |
| Kotlin callback 必须 `NewGlobalRef` 持有 | local ref 函数返回即失效，callback 被 GC 后悬空 | 空指针崩溃 |
| `NewStringUTF` 创建的 jstring 必须 `DeleteLocalRef` | local ref 表有上限（512），长输出会栈溢出 | `JNI ERROR (app bug): local reference table overflow` |
| 推理结束必须 `DetachCurrentThread` | 否则线程退出时 JVM 仍认为 active | 线程泄露 |
| 不在 callback 内调阻塞操作 | callback 在推理线程同步执行，阻塞会卡死推理 | 推理卡顿 |

### 2.4 参考实现

- **llama.android 官方模块**：`examples/llama.android/app/src/main/cpp/llama-android-jni.cpp` 的 `native_complete` 函数（流式 callback 范式）
- **ToolNeuron**：`service/inference/InferenceService.kt` 的 `streamComplete()` + 对应 native 文件 `InferenceClient.cpp` 的 `StreamCallback` 类

---

## 3. llama.cpp b9830 C API 清单（补强 ❌3）

### 3.1 wrapper 实际依赖的 C API

> 头文件：`include/llama.h`（b9830 版本，所有 API 均在该文件）

| C API | 用途 | wrapper 调用位置 | 参数要点 |
|---|---|---|---|
| `llama_model_load_from_file` | 从 GGUF 文件加载模型权重 | `loadModel()` | `path` + `params`（n_gpu_layers=0, use_mmap=true） |
| `llama_model_free` | 释放模型权重 | `unloadModel()` | 仅传 `llama_model*` |
| `llama_init_from_model` | 创建推理上下文（含 KV cache） | `loadModel()` | `model*` + `context_params`（n_ctx=2048, n_batch=512, n_threads=2） |
| `llama_free` | 释放推理上下文 | `unloadModel()` | 仅传 `llama_context*` |
| `llama_model_get_vocab` | 获取词表（用于 tokenize） | `complete()` / `streamComplete()` | 返回 `const llama_vocab*` |
| `llama_tokenize` | 把 prompt 字符串切为 token id 数组 | `complete()` / `streamComplete()` | `vocab*` + `text` + `text_len` + `tokens[]` + `n_tokens_max` + `add_special` + `parse_special` |
| `llama_sampler_chain_init` | 初始化采样器链（temp + top-p + top-k） | `complete()` / `streamComplete()` | `llama_sampler_chain_params`（`no_perf=true`） |
| `llama_sampler_chain_add` | 追加采样器（temp/top_p/top_k） | 同上 | `llama_sampler_init_temp(temp)` / `llama_sampler_init_top_p(p)` / `llama_sampler_init_top_k(k)` |
| `llama_sampler_sample` | 从 logits 采样下一个 token | 推理循环 | `sampler*` + `ctx*` + `batch` |
| `llama_sampler_free` | 释放采样器 | 推理结束 | `sampler*` |
| `llama_batch_get_one` | 构造单 token 输入 batch | 推理循环首步 | `tokens*` + `n_tokens` |
| `llama_decode` | 执行一次前向推理 | 推理循环 | `ctx*` + `batch` |
| `llama_get_logits` | 获取最后一 token 的 logits | 采样前 | `ctx*`（返回 `float*`，长度为 `n_vocab`） |
| `llama_token_to_piece` | token id 转字符串片段 | 流式 callback | `vocab*` + `llama_token` + `buf[]` + `length` + `lstrip` |
| `llama_token_is_eog` | 判断是否 end-of-generation token | 推理循环终止 | `vocab*` + `llama_token` |
| `llama_kv_self_clear` | 清空 KV cache（切换对话轮次时） | 多轮对话换轮时 | `ctx*` |
| `llama_backend_init` | 初始化 ggml 后端 | `JNI_OnLoad` | 无参 |
| `llama_backend_free` | 释放 ggml 后端 | `JNI_OnUnload` | 无参 |

### 3.2 推理循环伪代码

```cpp
// streamComplete 的核心循环
llama_batch batch = llama_batch_get_one(tokens, n_tokens);
while (true) {
    int n_ctx = llama_n_ctx(ctx);
    int n_used = llama_kv_self_seq_pos_max(ctx, 0);
    if (n_used + batch.n_tokens > n_ctx) {
        // 超出 ctx 窗口，清 KV cache（或 shift）
        llama_kv_self_clear(ctx);
    }

    if (llama_decode(ctx, batch) != 0) {
        // 推理失败（OOM / 内部错）
        break;
    }

    // 采样下一个 token
    llama_token new_token = llama_sampler_sample(sampler, ctx, batch.n_tokens - 1);
    if (llama_token_is_eog(vocab, new_token)) {
        break;  // EOS
    }

    // 转字符串并回调 Kotlin
    char buf[128];
    int len = llama_token_to_piece(vocab, new_token, buf, sizeof(buf), 0, false);
    if (len > 0) {
        std::string piece(buf, len);
        on_token_callback(piece.c_str(), nullptr);
    }

    // 用新 token 构造下一轮 batch
    batch = llama_batch_get_one(&new_token, 1);
}
```

### 3.3 b9830 API 变更注意（vs 旧版本）

| 旧 API（b1xxx 前） | b9830 新 API | 说明 |
|---|---|---|
| `llama_model_load` | `llama_model_load_from_file` | 参数从 `(path, params)` 改为 `(path, params)`，函数名变更 |
| `llama_new_context_with_model` | `llama_init_from_model` | b9830 重命名，参数同 |
| `llama_get_logits_ith` | `llama_get_logits_ith`（保留） | b9830 仍可用，但建议用 `llama_get_logits` 取末位 |
| `llama_token_eos` | `llama_token_is_eog` | b9830 改用 vocab + token 判断 |
| `llama_token_to_str` | `llama_token_to_piece` | b9830 改用 piece + buf 模式（避免 UTF-8 截断） |
| `llama_reset_timings` | 已废弃 | b9830 改用 `llama_perf_context_reset` |

**锁版本到 b9830**（[D7](../DECISIONS.md)）即可避免 API 漂移；`LlamaEngine.getVersion()` 启动时校验（[03 §3.3](./03_architecture_detail.md)）。

---

## 4. KSU 模块脚本完整草案（补强 ❌4 + ❌5）

### 4.1 module.prop 完整字段

```ini
# ksu_module/module.prop
id=mibrain
name=MiBrain
version=v0.1.0
versionCode=1
author=qbjsdsb
description=MiBrain KSU 模块：为 com.mibrain 配置 RECORD_AUDIO appops + 电池白名单。不含 APK 与模型，需另装。
```

### 4.2 post-fs-data.sh 完整草案

```bash
#!/system/bin/sh
# ksu_module/post-fs-data.sh
# MiBrain KSU 模块：在 post-fs-data 阶段配置 appops + 电池白名单
# 注意：post-fs-data.sh 执行时机极早（T=10s），此时 MiBrain APK 可能未安装
# 因此本脚本采用"探测 + graceful skip"模式，不阻塞启动

MODDIR="${0%/*}"
LOG_FILE="$MODDIR/install.log"
MIBRAIN_PKG="com.mibrain"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') $1" >> "$LOG_FILE"
}

log "=== post-fs-data.sh start ==="

# 1. 探测 MiBrain APK 是否已安装
MIBRAIN_UID=$(pm list packages -U "$MIBRAIN_PKG" 2>/dev/null | awk '{print $2}' | cut -d: -f2)

if [ -z "$MIBRAIN_UID" ]; then
    log "WARN: MiBrain APK 未安装，跳过 appops 配置（用户需先装 APK 再装 KSU 模块）"
    log "=== post-fs-data.sh end (skipped) ==="
    exit 0
fi

log "MiBrain UID = $MIBRAIN_UID"

# 2. 配置 appops（绕过 HyperOS 后台录音限制）
#    幂等：重复执行不报错
set_appops() {
    local op="$1"
    if appops set "$MIBRAIN_UID" "$op" allow 2>/dev/null; then
        log "OK: appops set $MIBRAIN_UID $op allow"
    else
        log "WARN: appops set $MIBRAIN_UID $op failed（HyperOS 3 可能不支持此 op）"
    fi
}

set_appops RECORD_AUDIO
set_appops OP_RECORD_AUDIO        # HyperOS 自定义 op（部分版本）
set_appops OP_POST_NOTIFICATIONS  # 前台通知（API 33+）

# 3. 电池白名单 + 关闭应用智能休眠（HyperOS 2.0+）
#    通过 sysconfig XML 配置（见 system/etc/sysconfig/mibrain.xml），此处仅兜底
dumpsys deviceidle whitelist "+$MIBRAIN_PKG" 2>/dev/null && \
    log "OK: deviceidle whitelist +$MIBRAIN_PKG" || \
    log "WARN: deviceidle whitelist failed"

# 4. 关闭应用智能休眠（HyperOS 3 独有，IPowerManager 内部）
settings put global settings_power_save_apps "" 2>/dev/null
# 注：mibrain.xml 的 allow-in-power 配置更可靠，此处仅兜底

log "=== post-fs-data.sh end ==="
exit 0
```

### 4.3 service.sh 决策（解决 T=10s APK 未装问题）

**问题**：post-fs-data.sh 在 T=10s 执行，此时 APK 可能未安装 → `pm list packages` 失败。

**决策**：不恢复 `service.sh`（[D7](../DECISIONS.md) 切回 JNI 后已删除），而是**在文档明确安装顺序约束**：

> **部署约束**（写入 [05_deploy_guide.md §0](./05_deploy_guide.md)）：用户必须**先装 MiBrain APK，再装 KSU 模块 zip**。若先装 KSU 模块，APK 启动后 `appops` 未配置，需手动 reboot 一次让 `post-fs-data.sh` 重跑。

**理由**：
1. `post-fs-data.sh` 的 graceful skip 设计已保证不阻塞启动
2. 用户首次安装时一定先有 APK（KSU 模块无 APK 无意义）
3. 恢复 `service.sh` 会引入"KSU 启动 root 子进程"的复杂度，违背 [D7](../DECISIONS.md) 简化原则

### 4.4 sepolicy.rule 完整规则草案

```cil
# ksu_module/sepolicy.rule
# MiBrain SELinux 放行规则（Phase 4 真机实测后修订）

# 1. 允许 MiBrain APK 进程访问 DE 加密区模型文件
#    （DE 区默认属于 untrusted_app 域，但 HyperOS 3 可能收紧）
allow untrusted_app mibrain_data_file:dir { search read write add_name remove_name };
allow untrusted_app mibrain_data_file:file { create open read write getattr setattr unlink };

# 2. 允许 MiBrain 加载未在 system 目录的 .so（dlopen libllama.so 等）
#    （Android 7+ 默认禁止 dlopen 非 system 路径的 .so，需放行）
allow untrusted_app apk_data_file:file execute;

# 3. 允许 MiBrain 通过 root 执行 am / cmd 命令（Phase 7 SystemSettingsTool）
allow untrusted_app shell_exec:file { execute read open getattr };

# 4. 允许 MiBrain 与 audio service 通信（绕过 HyperOS 录音限制的兜底）
allow untrusted_app audioserver_service:service_manager find;

# 5. 允许 MiBrain 读取 /proc/stat（CPU 温度监控，Phase 4 稳定性）
allow untrusted_app proc_stat:file { open read };

# 注：mibrain_data_file 是自定义 type，需在 file_contexts 中标记
# /data/user_de/0/com.mibrain/files(/.*)? u:object_r:mibrain_data_file:s0
```

### 4.5 system/etc/sysconfig/mibrain.xml（电池白名单）

```xml
<!-- ksu_module/system/etc/sysconfig/mibrain.xml -->
<?xml version="1.0" encoding="utf-8"?>
<config>
    <!-- 允许 MiBrain 在后台保活 -->
    <allow-association target="com.mibrain" />

    <!-- 电池白名单（HyperOS 3 + Android 15） -->
    <allow-in-power-save target="com.mibrain" />
    <allow-in-power-save-except-idle target="com.mibrain" />

    <!-- 允许前台服务保活 -->
    <allow-implicit-broadcast target="com.mibrain" />
</config>
```

---

## 5. sherpa-onnx 集成规范（补强 ⚠️6）

### 5.1 AAR 引入方式

```kotlin
// app/build.gradle.kts
dependencies {
    // sherpa-onnx 不在 Maven Central，必须本地 AAR 引入
    // AAR 来源：scripts/download_sherpa_aar.sh 下载到 app/libs/
    implementation(files("libs/sherpa-onnx-1.13.3.aar"))

    // 注：v1.13.3 仅有 git tag，无正式 GitHub Release notes
    // AAR 内含 jniLibs/arm64-v8a/libsherpa-onnx-jni.so + libonnxruntime.so
}
```

### 5.2 sherpa-onnx Kotlin API 真实类名表

> 修正 [14_feasibility_recheck_and_plan.md](./14_feasibility_recheck_and_plan.md) §三中误用的 `StreamingAsr`（设计稿自创词，sherpa-onnx 实际无此类）

| 能力 | Kotlin 真实类名 | Config 数据类 | 用途 |
|---|---|---|---|
| **流式 ASR** | `OnlineRecognizer` | `OnlineRecognizerConfig` + `OnlineRecognizerModelConfig`（含 `Transducer` / `Paraformer` / `Whisper` 子配置） | 实时语音转文字（`acceptWaveform(samples)` + `getResult()`） |
| **离线 ASR**（备选） | `OfflineRecognizer` | `OfflineRecognizerConfig` + `OfflineRecognizerModelConfig` | 一次性整段识别（不用于 MVP） |
| **TTS** | `OfflineTts` | `OfflineTtsConfig` + `OfflineTtsModelConfig` | 文字转语音（`generate(text)` → `OfflineTtsResult`，**无流式 chunk 回调**，[00 §4.3](./00_design_overview.md)） |
| **VAD** | `VoiceActivityDetector` | `VoiceActivityDetectorConfig`（含 `SileroVadModel`） | 检测说话段（`acceptWaveform(samples)` → `isEmpty()` / `isSpeech()`） |
| **KWS** | `KeywordSpotter` | `KeywordSpotterConfig` + `KeywordSpotterModelConfig` | 唤醒词检出（`acceptWaveform(samples)` → `getResult()`） |

### 5.3 AudioRecord 帧大小规范

```kotlin
// AudioRecord 配置
val sampleRate = 16000          // 16kHz（sherpa-onnx 要求）
val channelConfig = CHANNEL_IN_MONO
val audioFormat = ENCODING_PCM_16BIT

// 帧大小：30ms（480 samples），平衡延迟与 CPU 占用
// - 10ms 太频繁（100fps，CPU 占用高）
// - 100ms 延迟太高（唤醒检出延迟 100ms）
// - 30ms 是 sherpa-onnx 官方示例推荐值
val frameSizeInMs = 30
val frameSize = (sampleRate * frameSizeInMs / 1000)  // = 480 samples

// 喂给 sherpa-onnx
val samples = ShortArray(frameSize)
audioRecord.read(samples, 0, frameSize)
// sherpa-onnx 接受 ShortArray，内部转 float
onlineRecognizer.acceptWaveform(samples, sampleRate)
```

### 5.4 Direct Boot 下加载方案（PoC-A 细化）

**关键结论**：sherpa-onnx 内部用 `fopen` 读 ONNX 模型文件（绝对路径），与 Android `Context.createDeviceProtectedStorageContext()` 无关。**仅需 APK 自身进程有 DE 区访问权**。

```kotlin
// MiBrainForegroundService.kt（Direct Boot 模式下启动）
class MiBrainForegroundService : Service() {

    override fun onCreate() {
        super.onCreate()
        // Direct Boot 下用 DE Context
        val deContext = createDeviceProtectedStorageContext()

        // 模型路径用绝对路径（sherpa-onnx 内部 fopen）
        val modelDir = "/data/user_de/0/com.mibrain/files/models/"

        // 1. 检查模型是否存在
        val asrModelDir = File("${modelDir}sherpa-streaming-zipformer-bilingual-zh-en")
        if (!asrModelDir.exists()) {
            // 进入"等待模型下载"状态（详见 03 §2.3）
            return
        }

        // 2. 初始化 sherpa-onnx（直接传绝对路径，无需 openFileInput）
        val asrConfig = OnlineRecognizerConfig(
            modelConfig = OnlineRecognizerModelConfig(
                transducer = OnlineTransducerModelConfig(
                    encoder = "${asrModelDir}/encoder-epoch-99-avg-1.onnx",
                    decoder = "${asrModelDir}/decoder-epoch-99-avg-1.onnx",
                    joiner = "${asrModelDir}/joiner-epoch-99-avg-1.onnx",
                    tokens = "${asrModelDir}/tokens.txt"
                )
            )
        )
        val onlineRecognizer = OnlineRecognizer(asrConfig)

        // 3. System.loadLibrary 在 Direct Boot 下可加载（PoC-A 子项 1）
        //    因 APK 的 classLoader 在 Direct Boot 下仍可用
        //    （仅需 APK 声明 android:directBootAware="true"）

        // 4. ORT 是否调 SharedPreferences？实测确认（PoC-A 子项 3）
        //    sherpa-onnx v1.13.3 的 ORT 配置默认不写 SharedPreferences
        //    （但需在 Stage 1 PoC-A 真机验证）
    }
}
```

### 5.5 sherpa-onnx 资源生命周期表（与状态机对齐）

| 状态 | 进入时 acquire | 退出时 release |
|---|---|---|
| IDLE | VAD + KWS 已常驻（在 Service.onCreate 时创建） | - |
| LISTENING | `OnlineRecognizer`（lazy create，~200MB） | 转出 THINKING 时 release（省内存） |
| THINKING | LLM 推理（LlamaEngine，~1.3GB 常驻） | - |
| SPEAKING | `OfflineTts`（lazy create，~150MB） | 转出 COOLDOWN 时 release |
| COOLDOWN | - | - |
| TOOL_RUNNING | - | - |
| IMAGE_CAPTURING | - | - |
| PAUSED | VAD + KWS 仍常驻（可恢复） | LLM 可 unload（5min 超时后，[13 §6.3](./13_phase10_ux_enhancements_design.md)） |
| PROFILE_SWITCHING | unload 当前 TTS + KWS | load 新 profile 的 TTS + KWS |

**内存优化点**：ASR/TTS 按需创建/释放可省 350MB（200+150），对 8GB 设备的 7.5GB 峰值有显著缓解（[03 §6](./03_architecture_detail.md) 已计入此优化）。

---

## 6. 统一错误处理设计（补强 ⚠️7）

### 6.1 MiBrainError sealed class

```kotlin
// app/src/main/java/com/mibrain/core/error/MiBrainError.kt
package com.mibrain.core.error

/**
 * MiBrain 统一错误基类
 * 所有模块的错误必须继承此类，不允许直接抛 RuntimeException
 */
sealed class MiBrainError(
    val code: ErrorCode,
    override val message: String,
    override val cause: Throwable? = null
) : RuntimeException(message, cause)

// === 模型层错误 ===
class ModelNotFoundError(val modelPath: String) :
    MiBrainError(ErrorCode.MODEL_NOT_FOUND, "模型文件不存在: $modelPath")

class ModelLoadFailedError(val reason: String) :
    MiBrainError(ErrorCode.MODEL_LOAD_FAILED, "模型加载失败: $reason")

class ModelSha256MismatchError(val expected: String, val actual: String) :
    MiBrainError(ErrorCode.MODEL_SHA256_MISMATCH, "模型 SHA256 不匹配: expected=$expected, actual=$actual")

// === JNI 层错误 ===
class JniLoadFailedError(val libName: String) :
    MiBrainError(ErrorCode.JNI_LOAD_FAILED, "JNI 库加载失败: $libName")

class JniCallFailedError(val method: String, val nativeMsg: String) :
    MiBrainError(ErrorCode.JNI_CALL_FAILED, "JNI 调用失败: $method, native=$nativeMsg")

class JniOomError :
    MiBrainError(ErrorCode.JNI_OOM, "JNI native 内存不足（llama.cpp 推理 OOM）")

// === sherpa-onnx 错误 ===
class AsrFailedError(val reason: String) :
    MiBrainError(ErrorCode.ASR_FAILED, "ASR 推理失败: $reason")

class TtsFailedError(val reason: String) :
    MiBrainError(ErrorCode.TTS_FAILED, "TTS 合成失败: $reason")

class VadFailedError(val reason: String) :
    MiBrainError(ErrorCode.VAD_FAILED, "VAD 检测失败: $reason")

class KwsFailedError(val reason: String) :
    MiBrainError(ErrorCode.KWS_FAILED, "KWS 唤醒词检测失败: $reason")

// === 系统环境错误 ===
class LSPosedNotEnabledError :
    MiBrainError(ErrorCode.LSPOSED_NOT_ENABLED, "LSPosed Vector 框架未启用（/data/adb/lspd 不存在）")

class PhantomMicNotEnabledError :
    MiBrainError(ErrorCode.PHANTOM_MIC_NOT_ENABLED, "Phantom Mic 模块未启用（D14 风险）")

class ZygiskNotEnabledError :
    MiBrainError(ErrorCode.ZYGISK_NOT_ENABLED, "ZygiskNext 未启用（/data/adb/modules/zygisksu 不存在）")

// === 状态机错误 ===
class StateTransitionFailedError(val from: String, val to: String) :
    MiBrainError(ErrorCode.STATE_TRANSITION_FAILED, "状态转换失败: $from → $to（CAS 失败）")

class StateDeadlockDetectedError(val state: String, val stuckMs: Long) :
    MiBrainError(ErrorCode.STATE_DEADLOCK_DETECTED, "状态死锁: $state 卡住 ${stuckMs}ms")

// === 工具错误（Phase 6/7/9） ===
class ToolTimeoutError(val toolName: String, val timeoutMs: Long) :
    MiBrainError(ErrorCode.TOOL_TIMEOUT, "$toolName 超时 ${timeoutMs}ms")

class ToolNetworkError(val toolName: String, val httpCode: Int) :
    MiBrainError(ErrorCode.TOOL_NETWORK_ERROR, "$toolName 网络错误: HTTP $httpCode")

class ToolApiError(val toolName: String, val apiError: String) :
    MiBrainError(ErrorCode.TOOL_API_ERROR, "$toolName API 错误: $apiError")

class ToolParseError(val toolName: String, val rawResponse: String) :
    MiBrainError(ErrorCode.TOOL_PARSE_ERROR, "$toolName 响应解析失败: $rawResponse")

// === 系统压力错误 ===
class ApkKilledByMiuiError :
    MiBrainError(ErrorCode.APK_KILLED_BY_MIUI, "APK 被 HyperOS 杀进程（ForegroundService 死亡）")

class OnTrimMemoryCriticalError(val level: Int) :
    MiBrainError(ErrorCode.ON_TRIM_MEMORY_CRITICAL, "系统内存压力: level=$level")
```

### 6.2 ErrorCode 枚举

```kotlin
// app/src/main/java/com/mibrain/core/error/ErrorCode.kt
package com.mibrain.core.error

enum class ErrorCode {
    // 模型层（0x01xx）
    MODEL_NOT_FOUND,
    MODEL_LOAD_FAILED,
    MODEL_SHA256_MISMATCH,
    MODEL_DOWNLOAD_FAILED,

    // JNI 层（0x02xx）
    JNI_LOAD_FAILED,
    JNI_CALL_FAILED,
    JNI_OOM,

    // sherpa-onnx 层（0x03xx）
    ASR_FAILED,
    TTS_FAILED,
    VAD_FAILED,
    KWS_FAILED,

    // 系统环境（0x04xx）
    LSPOSED_NOT_ENABLED,
    PHANTOM_MIC_NOT_ENABLED,
    ZYGISK_NOT_ENABLED,

    // 状态机（0x05xx）
    STATE_TRANSITION_FAILED,
    STATE_DEADLOCK_DETECTED,

    // 工具（0x06xx）
    TOOL_TIMEOUT,
    TOOL_NETWORK_ERROR,
    TOOL_API_ERROR,
    TOOL_PARSE_ERROR,

    // 系统压力（0x07xx）
    APK_KILLED_BY_MIUI,
    ON_TRIM_MEMORY_CRITICAL,
}
```

### 6.3 错误处理统一流程

```
任意模块抛 MiBrainError
    ↓
ConversationEngine 捕获
    ↓
按 ErrorCode 分类处理:
    ├── 可恢复（TOOL_TIMEOUT / TOOL_NETWORK_ERROR）→ TTS "查询失败" + 回 IDLE
    ├── 可降级（PHANTOM_MIC_NOT_ENABLED）→ 启用降级方案 + 提示用户
    ├── 需用户操作（LSPOSED_NOT_ENABLED / ZYGISK_NOT_ENABLED）→ 显示引导 UI
    ├── 需重启（JNI_OOM / APK_KILLED_BY_MIUI）→ 写标志 + 提示 reboot
    └── 不可恢复（STATE_DEADLOCK_DETECTED）→ 强制 reset 到 IDLE + 写 ERROR 日志
    ↓
所有错误都记入 /data/user_de/0/com.mibrain/files/logs/error.log
```

---

## 7. 状态机完整规范（补强 ⚠️8/9）

### 7.1 完整 9 状态枚举（修正 [03 §4](./03_architecture_detail.md) 7 状态）

```kotlin
// app/src/main/java/com/mibrain/core/state/ConversationState.kt
package com.mibrain.core.state

enum class ConversationState {
    IDLE,               // 等待唤醒
    LISTENING,          // 正在录音 ASR
    THINKING,           // LLM 推理中
    SPEAKING,           // TTS 播放中
    COOLDOWN,           // 播放完 500ms 冷却（防回声尾巴）
    TOOL_RUNNING,       // 联网/手机控制工具执行中（Phase 6/7）
    IMAGE_CAPTURING,    // 等待用户拍照（Phase 9）
    PAUSED,             // 全局暂停（Phase 10 G17，双击电源键触发）
    PROFILE_SWITCHING,  // 切换 profile 中（Phase 10 G1）
}
```

> **注**：PAUSED + PROFILE_SWITCHING 由 Phase 10 引入，但主状态机需在 Phase 1 预留枚举值，避免后续枚举插入破坏序列化兼容性。

### 7.2 状态超时统一表（汇总所有文档）

| 状态 | 最大停留 | 超时处理 | 来源 |
|---|---|---|---|
| IDLE | 无上限 | - | - |
| LISTENING | 30s | 强制 VAD 结束 → THINKING（避免长时间录音） | 本规范新增 |
| THINKING | 30s | TTS "超时" + 回 IDLE | [03 §4](./03_architecture_detail.md) |
| SPEAKING | 60s | 强制停止 AudioTrack → COOLDOWN | 本规范新增 |
| COOLDOWN | 1000ms（可配置） | → IDLE | [03 §4](./03_architecture_detail.md) |
| TOOL_RUNNING | 5s（Phase 6） / 30s（Phase 9 vision） | TTS "查询失败" + → IDLE（超时降级路径） | [09 §3](./09_phase6_network_tools_design.md) + 本规范 |
| IMAGE_CAPTURING | 60s | → IDLE + TTS "超时未拍照" | [03 §4](./03_architecture_detail.md) + [12 §3](./12_phase9_multimodal_design.md) |
| PAUSED | 5min | 自动回 IDLE（不卸载模型） | [13 §6.3](./13_phase10_ux_enhancements_design.md) |
| PROFILE_SWITCHING | 30s | 失败回滚原 profile | [13 §7](./13_phase10_ux_enhancements_design.md) |

### 7.3 超时降级路径（与主转移路径并存）

主转移路径（[03 §4](./03_architecture_detail.md)）：
```
TOOL_RUNNING → THINKING（工具成功）
```

超时降级路径（本规范新增，**合法**）：
```
TOOL_RUNNING → IDLE（超时 5s/30s 后，跳过 THINKING/SPEAKING/COOLDOWN 直接回 IDLE）
```

**实现**：`transitionTo(TOOL_RUNNING, IDLE)` 在超时 handler 中调用，CAS 成功后 TTS "查询失败"。

### 7.4 状态机 watchdog（防死锁）

```kotlin
// ConversationEngine.kt
private val stateEntryTime = AtomicLong(0)
private val watchdogHandler = Handler(HandlerThread("state-watchdog").looper)

fun transitionTo(expected: ConversationState, new: ConversationState): Boolean {
    val success = state.compareAndSet(expected, new)
    if (success) {
        stateEntryTime.set(System.currentTimeMillis())
        scheduleWatchdog(new)  // 安排超时检查
    }
    return success
}

private fun scheduleWatchdog(state: ConversationState) {
    val timeoutMs = when (state) {
        ConversationState.LISTENING -> 30_000L
        ConversationState.THINKING -> 30_000L
        ConversationState.SPEAKING -> 60_000L
        ConversationState.COOLDOWN -> 1_000L
        ConversationState.TOOL_RUNNING -> 30_000L  // Phase 9 vision 上限
        ConversationState.IMAGE_CAPTURING -> 60_000L
        ConversationState.PAUSED -> 300_000L
        ConversationState.PROFILE_SWITCHING -> 30_000L
        ConversationState.IDLE -> Long.MAX_VALUE  // 无超时
    }

    watchdogHandler.postDelayed({
        // 检查是否仍在该状态
        if (this.state.get() == state) {
            // 仍在原状态且超时 → 强制 reset
            val stuckMs = System.currentTimeMillis() - stateEntryTime.get()
            if (stuckMs > timeoutMs * 10) {
                // 超时 10 倍 → 死锁，强制 reset + 错误日志
                errorLogger.log(StateDeadlockDetectedError(state.name, stuckMs))
                state.set(ConversationState.IDLE)  // 强制 reset
                tts.speak("系统异常已重置")
            } else if (stuckMs > timeoutMs) {
                // 正常超时 → 走超时降级路径
                handleStateTimeout(state)
            }
        }
    }, timeoutMs)
}
```

### 7.5 多轮对话接续路径（[03 §4](./03_architecture_detail.md) 注释项正式化）

```
SPEAKING → COOLDOWN → IDLE                  # 单轮对话（默认）
SPEAKING → COOLDOWN → LISTENING             # 多轮对话（Phase 9，跳过唤醒）
```

**实现**：在 COOLDOWN 超时 handler 中检查 `conversationMode`：
- `SINGLE` → `transitionTo(COOLDOWN, IDLE)`
- `MULTI` → `transitionTo(COOLDOWN, LISTENING)`（直接接续听，省唤醒）

---

## 8. 遗留待补项（🟡 建议级，对应 Phase 实施时补）

以下 4 项不阻塞 Phase 1，但建议在对应 Phase 实施前补强：

| # | 待补项 | 影响阶段 | 优先补强时机 |
|---|---|---|---|
| 1 | 1500 行 JNI wrapper 工程量分解表（拆 8 个分项） | Phase 1 工作量估算 | Phase 1 启动前 |
| 2 | CI release workflow 蓝图文件（`.github/workflows/release.yml` + keystore 流程） | Phase 5 发布 | Phase 5 启动前 |
| 3 | Phase 6/7/9 子故障错误码细分表（WeatherTool/TranslateTool/WebSearchTool/NewsTool 各自的 4xx/5xx/timeout 处理；ScreenCaptureTool 失败根因分类；GLM-4V 401/403/429/500 处理） | Phase 6/7/9 实施 | 各 Phase 启动前 |
| 4 | APK 被 MIUI 杀后的检测 + 复活通知流程（BootReceiver 检测 `lastExitWasAbnormal` 标志 → 发本地通知引导用户重做白名单） | Phase 4 稳定性测试 | Phase 4 启动前 |

---

## 9. 引用关系

本文档被以下文件引用（2026-06-30 第五轮补强后实际状态）：

**已建立引用**：
- [README.md](../README.md) §修订说明 + §文档清单（审查与技术规范段） → 引用 §1-§8 概览
- [docs/README.md](./README.md) §文档清单 → 引用 §1-§8 概览
- [docs/00_design_overview.md](./00_design_overview.md) §11 Phase 1 路线图 → 引用 §1-§4（CMake + JNI + C API + KSU 脚本）
- [docs/03_architecture_detail.md](./03_architecture_detail.md) §2.2 启动顺序 → 引用 §4 KSU 模块脚本完整草案
- [docs/03_architecture_detail.md](./03_architecture_detail.md) §3 JNI 推理接口 → 引用 §1 CMake + §2 JNI 跨线程 callback + §3 C API 清单
- [docs/03_architecture_detail.md](./03_architecture_detail.md) §4 回环防护协议 → 引用 §6 错误处理 + §7 状态机完整规范

**待补引用**（Phase 1 启动前在对应文件加入）：
- [ksu_module/README.md](../ksu_module/README.md) → 引用 §4 KSU 模块脚本完整草案（module.prop / post-fs-data.sh / sepolicy.rule / sysconfig XML）
- [scripts/README.md](../scripts/README.md) → 引用 §1 build_llama_jni.sh 完整 CMake 命令 + §5 sherpa-onnx AAR 引入方式
- [app/src/main/jni/CMakeLists.txt] → 引用 §1.1 CMake 选项清单（Phase 1 编码时填入）
