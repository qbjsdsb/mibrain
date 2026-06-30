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
    -DLLAMA_BUILD_APP=OFF \
    -DGGML_OPENMP=OFF  # 默认 ON 会依赖 libomp.so，Android NDK 不自带（需另推 libomp.so 到 jniLibs），关闭后改用 std::thread，体积更小且无依赖风险

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
| `GGML_OPENMP` | `OFF` | 必备 | 默认 ON 依赖 `libomp.so`（NDK 不自带，需另推 jniLibs），关闭改用 `std::thread`，规避运行时 `dlopen libomp.so failed` |

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
| `llama_token_to_piece` | token id 转字符串片段 | 流式 callback | `vocab*` + `llama_token` + `buf[]` + `length` + `lstrip` + `special(bool)` |
| `llama_vocab_is_eog` | 判断是否 end-of-generation token | 推理循环终止 | `vocab*` + `llama_token`（b9830 改用 vocab 级判断，旧 `llama_token_is_eog` 已 DEPRECATED） |
| `llama_get_memory` | 获取 context 的 memory 句柄 | 调用 `llama_memory_*` 前 | `ctx*` → `llama_memory_t`（b9830：KV cache 由 `llama_memory_t` 抽象承载） |
| `llama_memory_clear` | 清空 KV cache（切换对话轮次时） | 多轮对话换轮时 | `llama_memory_t` + `bool`（需先 `llama_get_memory(ctx)` 取 mem；`true` 表示完全清零） |
| `llama_memory_seq_pos_max` | 获取序列已用位置 | 推理循环 ctx 检查 | `llama_memory_t` + `seq_id`（替代旧 `llama_kv_self_seq_pos_max`，b9830 已移除） |
| `llama_backend_init` | 初始化 ggml 后端 | `JNI_OnLoad` | 无参 |
| `llama_backend_free` | 释放 ggml 后端 | `JNI_OnUnload` | 无参 |

### 3.2 推理循环伪代码

```cpp
// streamComplete 的核心循环
llama_batch batch = llama_batch_get_one(tokens, n_tokens);
while (true) {
    int n_ctx = llama_n_ctx(ctx);
    int n_used = llama_memory_seq_pos_max(llama_get_memory(ctx), 0);
    if (n_used + batch.n_tokens > n_ctx) {
        // 超出 ctx 窗口，清 KV cache（或 shift）
        // b9830：KV cache 由 llama_memory_t 承载，旧 llama_kv_self_clear 已移除
        llama_memory_clear(llama_get_memory(ctx), true);
    }

    if (llama_decode(ctx, batch) != 0) {
        // 推理失败（OOM / 内部错）
        break;
    }

    // 采样下一个 token
    llama_token new_token = llama_sampler_sample(sampler, ctx, batch.n_tokens - 1);
    if (llama_vocab_is_eog(vocab, new_token)) {
        break;  // EOS（b9830：用 llama_vocab_is_eog，旧 llama_token_is_eog 已 DEPRECATED）
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
| `llama_token_eos` | `llama_vocab_is_eog` | b9830 改用 vocab 级判断（`llama_token_is_eog` 已 DEPRECATED） |
| `llama_kv_self_clear` / `llama_kv_self_seq_pos_max` | `llama_memory_clear` / `llama_memory_seq_pos_max` | b9830 将 KV cache 抽象为 `llama_memory_t`，需先 `llama_get_memory(ctx)` 取句柄 |
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

```magiskpolicy
# ksu_module/sepolicy.rule（KSU 使用 magiskpolicy rule 语法，非 CIL）
# MiBrain SELinux 放行规则（Phase 4 真机实测后修订）

# 0. 声明自定义 type（必须先声明再使用，否则 ksud 加载报 "unknown type mibrain_data_file"）
#    file_type：标记为文件 type；mlstrustedobject：允许跨域访问（untrusted_app 可读）
type mibrain_data_file
typeattribute mibrain_data_file file_type
typeattribute mibrain_data_file mlstrustedobject

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

# 注：mibrain_data_file 是自定义 type，必须在 file_contexts 中标记路径才能生效
# file_contexts 文件：ksu_module/system/etc/file_contexts/mibrain_file_contexts
# 内容：/data/user_de/0/com.mibrain/files(/.*)? u:object_r:mibrain_data_file:s0
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
| **流式 ASR** | `OnlineRecognizer` | `OnlineRecognizerConfig` + `OnlineModelConfig`（含 `Transducer` / `Paraformer` / `Whisper` 子配置） | 实时语音转文字（需 `createStream()` → `stream.acceptWaveform(samples, sr)` → `decode(stream)` → `getResult(stream)`，**不可直接 `onlineRecognizer.acceptWaveform()`**） |
| **离线 ASR**（备选） | `OfflineRecognizer` | `OfflineRecognizerConfig` + `OfflineModelConfig` | 一次性整段识别（不用于 MVP） |
| **TTS** | `OfflineTts` | `OfflineTtsConfig` + `OfflineTtsModelConfig` | 文字转语音（`generate(text)` → `GeneratedAudio`；另提供 `generateWithCallback(...)` 进度回调用于流式播放，[00 §4.3](./00_design_overview.md)） |
| **VAD** | `Vad` | `VadModelConfig`（含 `SileroVadModelConfig`） | 检测说话段（`acceptWaveform(samples, sr)` → `empty()` / `isSpeechDetected()`，注意方法名非 `isEmpty`/`isSpeech`） |
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

// 喂给 sherpa-onnx（必须走 createStream 模式，不可直接调 recognizer.acceptWaveform）
val samples = ShortArray(frameSize)
audioRecord.read(samples, 0, frameSize)
// sherpa-onnx 接受 ShortArray，内部转 float
// stream 在 LISTENING 进入时一次性创建，退出时释放（与 [§5.5](#55-sherpa-onnx-资源生命周期表与状态机对齐) 生命周期表对齐）
stream.acceptWaveform(samples, sampleRate)
if (onlineRecognizer.isReady(stream)) {
    onlineRecognizer.decode(stream)
}
val result = onlineRecognizer.getResult(stream)
// result.text 即增量转写文本
```

### 5.4 Direct Boot 下加载方案（PoC-A 细化）

**关键结论（待验证假设）**：sherpa-onnx 内部用 `fopen` 读 ONNX 模型文件（绝对路径），与 Android `Context.createDeviceProtectedStorageContext()` 无关，**仅需 APK 自身进程有 DE 区访问权**。
> ⚠️ **R1 排查发现**：Direct Boot + sherpa-onnx/onnxruntime 组合**无公开先例**（BCR 仅验证 AudioRecord，不覆盖 ONNX 模型加载）。onnxruntime 是否内部调 CE-only API（如 SharedPreferences/ContentProvider）尚未证实。**此结论必须由 PoC-A 真机验证**，验证前不得据此进入 Phase 2 实现。验证失败时的降级路径：模型改放 CE 区并放弃 Direct Boot 锁屏唤醒（详见 [DECISIONS.md D14](../DECISIONS.md) 降级方案）。

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
            modelConfig = OnlineModelConfig(
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

// === native crash（第六轮 R3 新增；进程仍会死，先落 crash dump 供下次启动诊断） ===
class NativeSigsegvError(val addr: String) :
    MiBrainError(ErrorCode.NATIVE_SIGSEGV, "native SIGSEGV 段错误: $addr")

class NativeSigabrtError(val nativeMsg: String) :
    MiBrainError(ErrorCode.NATIVE_SIGABRT, "native SIGABRT: $nativeMsg")

class NativeSigbusError(val addr: String) :
    MiBrainError(ErrorCode.NATIVE_SIGBUS, "native SIGBUS（mmap/对齐）: $addr")

class NativeJniFatalError(val nativeMsg: String) :
    MiBrainError(ErrorCode.NATIVE_JNI_FATAL, "JNI FatalError: $nativeMsg")

class NativeStackOverflowError :
    MiBrainError(ErrorCode.NATIVE_STACK_OVERFLOW, "native 栈溢出（深递归 token callback）")
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

    // native crash（0x08xx，第六轮 R3 新增；信号处理器捕获，进程仍会死但先落 crash dump）
    NATIVE_SIGSEGV,           // SIGSEGV 段错误（悬空指针/越界）
    NATIVE_SIGABRT,           // SIGABRT（llama.cpp assert / std::abort）
    NATIVE_SIGBUS,            // SIGBUS（mmap 模型文件失败 / 对齐错误）
    NATIVE_JNI_FATAL,         // JNI FatalError（env->FatalError 主动 abort）
    NATIVE_STACK_OVERFLOW,    // native 栈溢出（深递归 token callback 链）
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

### 6.4 native crash 处理（第六轮 R3 新增）

native crash（SIGSEGV/SIGABRT/SIGBUS/栈溢出）**无法被 Kotlin try-catch 捕获**，进程必死。处理目标是"死前留证据 + 下次启动自愈"：

**1. 信号处理器（JNI_OnLoad 注册 sigaction）**：
```cpp
// llama_engine_jni.cpp
static void write_crash_dump(int sig, siginfo_t* info, void* ctx) {
    // 写 crash dump 到 DE 区（Direct Boot 可读，下次启动诊断）
    FILE* f = fopen("/data/user_de/0/com.mibrain/files/logs/native_crash.log", "a");
    if (f) {
        fprintf(f, "[%ld] sig=%d addr=%p code=%d\n",
                time(nullptr), sig, info->si_addr, info->si_code);
        // 附带 llama.cpp 版本 + 当前状态（通过全局 g_state_str）
        fclose(f);
    }
    // 重置默认 handler 后 re-raise，让 tombstone 正常生成
    signal(sig, SIG_DFL);
    raise(sig);
}

JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved) {
    g_jvm = vm;
    struct sigaction sa{};
    sa.sa_sigaction = write_crash_dump;
    sa.sa_flags = SA_SIGINFO;
    sigaction(SIGSEGV, &sa, nullptr);
    sigaction(SIGABRT, &sa, nullptr);
    sigaction(SIGBUS, &sa, nullptr);
    // SIGPIPE 忽略（sherpa-onnx/onnxruntime 内部可能触发）
    signal(SIGPIPE, SIG_IGN);
    // ...
}
```

**2. 下次启动自愈（BootReceiver / Service.onCreate 检测）**：
- 启动时读 `native_crash.log`，若末尾时间戳距今 < 5min → 标记 `lastExitWasAbnormal = true`
- 写 `ErrorCode.NATIVE_*` 到 error.log，TTS 提示"上次异常退出，已重置"
- **降级**：若同一信号在 1h 内重复 ≥ 3 次 → 禁用对应能力（如 SIGSEGV 在 streamComplete 期间 → 禁用 LLM，仅保留 ASR/TTS 读通知）

**3. crash 边界**：
| 信号 | 常见根因 | 降级动作 |
|---|---|---|
| SIGSEGV | 悬空 JNI 全局引用（callback 被 GC）/ KV cache 句柄失效 | 检查 NewGlobalRef 生命周期；降级禁用流式 |
| SIGABRT | llama.cpp assert（ctx 超限）/ std::bad_alloc | 缩小 n_ctx；降级 1.5B→不加载 |
| SIGBUS | mmap 模型文件失败（DE 区 Direct Boot 未就绪） | 改用 fread 而非 mmap；延迟到解锁后加载 |
| 栈溢出 | token callback 链过深（递归调 streamComplete） | 限制 callback 不调推理；改用队列异步 |

### 6.5 资源泄漏防护清单（第六轮 R3 新增）

每个 native/系统资源必须有明确 acquire/release 配对，且在状态机异常退出（watchdog 强制 reset / native crash）时仍能释放：

| # | 资源 | acquire 时机 | release 时机 | 异常退出兜底 | 违反后果 |
|---|---|---|---|---|---|
| 1 | `Vad`（sherpa-onnx） | Service.onCreate | Service.onDestroy | Service.onDestroy 必被系统调（即使 crash，进程死时触发） | VAD 模型常驻内存泄漏，重启失效 |
| 2 | `KeywordSpotter`（KWS） | Service.onCreate | Service.onDestroy | 同上 | KWS 失效无法唤醒 |
| 3 | `AudioRecord` | LISTENING 进入 | LISTENING 退出（→THINKING） | watchdog 超时 handler 内强制 `audioRecord.stop()+release()` | 麦克风被占，其他 app 静音 |
| 4 | `PowerManager.WakeLock` | IDLE→LISTENING | COOLDOWN→IDLE | watchdog 强制 reset 时在 reset 路径里 `if (wakeLock.isHeld()) wakeLock.release()` | 设备无法休眠，耗电 |
| 5 | `JNIEnv*`（AttachCurrentThread） | native 推理线程进入 | 推理结束 detach | 见 §2.3 约束表；crash 时线程直接死，JVM 进程死，无泄漏累积 | 线程泄露，JVM active thread 计数溢出 |
| 6 | `jobject`（NewGlobalRef） | streamComplete 入口 | streamComplete 出口 DeleteGlobalRef | crash 时进程死，全局表随进程消失；但**正常路径若漏 DeleteGlobalRef → 每次调用累积 1 个**，长跑必溢出 | local/global ref 表溢出崩溃 |
| 7 | `Ort::Session`（onnxruntime，ASR/TTS lazy create） | LISTENING/SPEAKING 进入 | 转出时 release（[§5.5](#55-sherpa-onnx-资源生命周期表与状态机对齐)） | watchdog 超时 handler 内 `onlineRecognizer?.let { it.free(); onlineRecognizer = null }` | ONNX session 内存 ~200MB 不释放，OOM |

**实现约束**：
- 所有 acquire/release 必须在 `try/finally` 中成对，`finally` 内 release 前判空
- watchdog 强制 reset 路径必须遍历上表 #3/#4/#7（#1/#2 随 Service 生命周期，#5/#6 随推理线程）
- 新增"资源泄漏检测"：debug 构建中，每次状态退出时断言对应资源已释放（`check(!wakeLock.isHeld())`），release 构建移除断言

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

> **第六轮 R3 修复 4 项代码级 bug**：
> 1. ~~`state.set(IDLE)` 绕过 CAS~~ → 改用 `compareAndSet(stuckState, IDLE)`，避免覆盖合法转移导致推理结果丢失
> 2. ~~推理线程无法取消~~ → 新增 `cancelInference` 标志，token callback 检查后提前 break（llama_decode 阻塞无法硬中断，但可在 token 间检查点退出）
> 3. ~~watchdog 线程自身死了无人救~~ → 新增主线程 fallback 定期检查（watchdog-of-watchdog）
> 4. ~~onTrimMemory 调 unloadModel 与推理竞态~~ → 新增 `inferenceLock` 互斥，unloadModel 前等推理结束

```kotlin
// ConversationEngine.kt
private val stateEntryTime = AtomicLong(0)
private val watchdogHandler = Handler(HandlerThread("state-watchdog").looper)
private val cancelInference = AtomicBoolean(false)        // fix #2：推理取消标志
private val inferenceLock = ReentrantLock()               // fix #4：unloadModel 与推理互斥

fun transitionTo(expected: ConversationState, new: ConversationState): Boolean {
    check((expected to new) in legalTransitions) { "非法转移: $expected → $new" }  // [§7.6](#76-paused--profile_switching-完整转移路径第六轮-r3-补全原-10-条漏转移) 入边校验
    val success = state.compareAndSet(expected, new)
    if (success) {
        stateEntryTime.set(System.currentTimeMillis())
        scheduleWatchdog(new)
    }
    return success
}

private fun scheduleWatchdog(stuckState: ConversationState) {
    val timeoutMs = when (stuckState) {
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
        // fix #1：用 CAS 而非 set()，避免覆盖合法转移（如 token callback 恰在此时推进 THINKING→SPEAKING）
        if (state.get() == stuckState) {
            val stuckMs = System.currentTimeMillis() - stateEntryTime.get()
            if (stuckMs > timeoutMs * 10) {
                // 死锁：超时 10 倍 → 强制 reset（仍走 CAS，若 CAS 失败说明刚合法转移，放弃 reset）
                errorLogger.log(StateDeadlockDetectedError(stuckState.name, stuckMs))
                if (state.compareAndSet(stuckState, ConversationState.IDLE)) {  // fix #1：CAS 替代 set()
                    releaseStateResources(stuckState)  // [§6.5](#65-资源泄漏防护清单第六轮-r3-新增) 释放该态占用资源
                    tts.speak("系统异常已重置")
                }
            } else if (stuckMs > timeoutMs) {
                // 正常超时 → 走超时降级路径（内部仍用 CAS）
                if (stuckState == ConversationState.THINKING) {
                    cancelInference.set(true)  // fix #2：通知 token callback 提前 break
                }
                handleStateTimeout(stuckState)
            }
        }
    }, timeoutMs)
}

// fix #2：token callback 检查取消标志（native 推理循环每 token 调一次）
// 在 [§2.2](#22-规范实现) on_token_callback 开头加：
//   if (cancelInference.load()) { return_with_break; }  // native 侧需读 volatile 标志

// fix #3：watchdog-of-watchdog（主线程每 60s 兜底检查，防 watchdog HandlerThread 死）
fun startWatchdogOfWatchdog() {
    mainHandler.postDelayed(object : Runnable {
        override fun run() {
            val current = state.get()
            if (current != ConversationState.IDLE) {
                val stuckMs = System.currentTimeMillis() - stateEntryTime.get()
                if (stuckMs > 600_000L) {  // 10min 兜底（远大于各态 timeoutMs*10）
                    // watchdog 线程可能已死，主线程强制 reset
                    errorLogger.log(StateDeadlockDetectedError(current.name, stuckMs))
                    state.compareAndSet(current, ConversationState.IDLE)
                    releaseStateResources(current)
                    // 重启 watchdog HandlerThread
                    watchdogHandler.looper.thread.let { if (!it.isAlive) restartWatchdogThread() }
                }
            }
            mainHandler.postDelayed(this, 60_000L)
        }
    }, 60_000L)
}

// fix #4：onTrimMemory 与推理互斥
fun onTrimMemory(level: Int) {
    if (level >= TRIM_MEMORY_RUNNING_CRITICAL) {
        inferenceLock.lock()  // 等推理结束（若正在 THINKING）
        try {
            if (state.get() != ConversationState.THINKING) {  // 双重检查，避免锁期间进入 THINKING
                llamaEngine.unloadModel()  // onTrimMemoryCriticalError 降级
            }
        } finally {
            inferenceLock.unlock()
        }
    }
}
// 推理入口同样加锁：fun streamComplete(...) = inferenceLock.withLock { ... native 推理 ... }
```

**资源释放辅助（[§6.5](#65-资源泄漏防护清单第六轮-r3-新增) 落地）**：
```kotlin
private fun releaseStateResources(stuckState: ConversationState) {
    // 遍历 §6.5 表中需在异常退出释放的资源（#3/#4/#7）
    if (stuckState == ConversationState.LISTENING) {
        audioRecord?.stop(); audioRecord?.release(); audioRecord = null
    }
    if (wakeLock.isHeld()) wakeLock.release()
    onlineRecognizer?.let { it.free(); onlineRecognizer = null }  // ASR session
    offlineTts?.let { it.free(); offlineTts = null }               // TTS session
    cancelInference.set(false)  // 重置取消标志，供下次推理
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

### 7.6 PAUSED / PROFILE_SWITCHING 完整转移路径（第六轮 R3 补全，原 10 条漏转移）

> **R3 排查发现**：PAUSED + PROFILE_SWITCHING 在 [§7.1](#71-完整-9-状态枚举修正-03-4-7-状态) 已声明，但入边（哪些状态可进入）和退出目标完全未定义，属"半死状态"。本节正式补全。

**PAUSED（Phase 10 G17 全局暂停，双击电源键触发）**：

```
入边（7 条，任意运行态都可暂停，但需先释放该态占用的资源）:
  IDLE           --(双击电源键)--> PAUSED            # 无需释放
  LISTENING      --(双击电源键)--> PAUSED            # 先 audioRecord.stop()+release()
  THINKING       --(双击电源键)--> PAUSED            # 设 cancelInference 标志，token callback 检测后提前 break（见 §7.4 watchdog fix #2）
  SPEAKING       --(双击电源键)--> PAUSED            # 先 audioTrack.stop()+release()
  COOLDOWN       --(双击电源键)--> PAUSED            # 无需释放
  TOOL_RUNNING   --(双击电源键)--> PAUSED            # 先取消工具（工具自身的 cancel）
  IMAGE_CAPTURING--(双击电源键)--> PAUSED            # 先取消拍照等待

出边（2 条）:
  PAUSED --(再次双击电源键)--> IDLE                  # 恢复，不恢复之前被打断的对话（简化）
  PAUSED --(5min 超时)--> IDLE                        # 自动恢复省电（[§7.2](#72-状态超时统一表汇总所有文档)）
```

**约束**：
- PAUSED 期间 VAD + KWS **保持常驻**（[§5.5](#55-sherpa-onnx-资源生命周期表与状态机对齐)），但 AudioRecord 停止喂帧（不浪费 CPU）
- PAUSED 期间 LLM 可按 keep-alive 策略卸载（5min 超时后，[03 §3.4](./03_architecture_detail.md)）
- THINKING→PAUSED 的推理中断**非即时**：llama_decode 是阻塞调用无法中断，cancel 标志在下一个 token callback 检查点生效（最坏延迟 = 单 token 推理时间 ~100ms），可接受

**PROFILE_SWITCHING（Phase 10 G1 多用户/profile 切换）**：

```
入边（1 条，仅 IDLE 可切，避免中断进行中的对话）:
  IDLE --(用户在 UI 切 profile)--> PROFILE_SWITCHING

出边（2 条）:
  PROFILE_SWITCHING --(切换成功)--> IDLE             # load 新 profile 的 TTS + KWS，旧 profile 资源已 unload
  PROFILE_SWITCHING --(切换失败 / 30s 超时)--> IDLE  # 回滚原 profile（unload 半成品 + reload 原 profile）
```

**约束**：
- 仅 IDLE→PROFILE_SWITCHING 合法（LISTENING/THINKING/SPEAKING 中切 profile 会丢失当前对话，禁止）；UI 在非 IDLE 态禁用切换按钮
- PROFILE_SWITCHING 期间 unload 当前 TTS + KWS（[§5.5](#55-sherpa-onnx-资源生命周期表与状态机对齐)），VAD 可保留（VAD 与 profile 无关）
- 切换失败回滚必须保证 TTS + KWS 至少有一份可用（原 profile 或新 profile），不允许两个都 unload 后卡死

**入边合法性校验（ConversationEngine.transitionTo 前置检查）**：
```kotlin
private val legalTransitions = setOf(
    IDLE to LISTENING, IDLE to SPEAKING, IDLE to TOOL_RUNNING, IDLE to PAUSED, IDLE to PROFILE_SWITCHING,
    LISTENING to THINKING, LISTENING to PAUSED,
    THINKING to SPEAKING, THINKING to TOOL_RUNNING, THINKING to PAUSED,
    TOOL_RUNNING to THINKING, TOOL_RUNNING to IMAGE_CAPTURING, TOOL_RUNNING to IDLE, TOOL_RUNNING to PAUSED,
    IMAGE_CAPTURING to TOOL_RUNNING, IMAGE_CAPTURING to IDLE, IMAGE_CAPTURING to PAUSED,
    SPEAKING to COOLDOWN, SPEAKING to PAUSED,
    COOLDOWN to IDLE, COOLDOWN to LISTENING, COOLDOWN to PAUSED,
    PAUSED to IDLE,
    PROFILE_SWITCHING to IDLE,
)
fun transitionTo(expected: ConversationState, new: ConversationState): Boolean {
    check((expected to new) in legalTransitions) { "非法转移: $expected → $new" }
    // ... CAS + watchdog
}
```

---

## 8. 遗留待补项（🟡 建议级，对应 Phase 实施时补）

以下 4 项不阻塞 Phase 1，但建议在对应 Phase 实施前补强：

| # | 待补项 | 影响阶段 | 优先补强时机 |
|---|---|---|---|
| 1 | 1500 行 JNI wrapper 工程量分解表（拆 8 个分项） | Phase 1 工作量估算 | Phase 1 启动前 |
| 2 | CI release workflow 蓝图文件（`.github/workflows/release.yml` + keystore 流程） | Phase 5 发布 | Phase 5 启动前 |
| 3 | Phase 6/7/9 子故障错误码细分表（WeatherTool/TranslateTool/WebSearchTool/NewsTool 各自的 4xx/5xx/timeout 处理；ScreenCaptureTool 失败根因分类；GLM-4V 401/403/429/500 处理） | Phase 6/7/9 实施 | 各 Phase 启动前 |
| 4 | APK 被 MIUI 杀后的检测 + 复活通知流程（BootReceiver 检测 `lastExitWasAbnormal` 标志 → 发本地通知引导用户重做白名单） | Phase 4 稳定性测试 | Phase 4 启动前 |

### 8.1 第六轮 R1-R6 排查遗留 19 项汇总（跨文档索引）

> 第六轮全维度排查（R1-R6）后，b1-b4 批次已修复全部 ❌ 阻塞级 + ⚠️ 强烈建议级条目。以下 19 项为仍遗留的 🟡 建议级 / "对应 Phase 实施时补" / "未来" / "Phase 11+" 项，按维度归并。其中 §8 表 #1-#4 与本表 #7/#11/#12/#17 为同一项（跨文档交叉引用），不重复计数；本表作用是提供**单一权威索引**，避免遗漏。

| # | 维度 | 来源（文件:行号） | 内容摘要 | 影响 Phase | 补强时机 |
|---|---|---|---|---|---|
| 1 | R1 | [15 §1.5](#15-phase-11-gpu-加速时新增选项未来仅记录) | Phase 11+ GPU 加速 CMake 选项（OpenCL/Hexagon，仅记录） | Phase 11+ | Phase 11+ 启动时 |
| 2 | R1 | [15 §1.5](#15-phase-11-gpu-加速时新增选项未来仅记录) | OpenCL 与 sherpa-onnx onnxruntime backend 冲突需实测 | Phase 11+ | Phase 11+ 启动时 |
| 3 | R1 | [DECISIONS.md D7](../DECISIONS.md) | Phase 1 启动前复查 llama.cpp 是否有更新稳定版（b9844+） | Phase 1 | Phase 1 启动前 |
| 4 | R2 | [DECISIONS.md D26](../DECISIONS.md) | Phase 10 G6 字幕同步需 Kotlin 端按 200ms 切 PCM 成 chunk 喂 AudioTrack | Phase 9/10 | Phase 10 启动前 |
| 5 | R2 | [15 §5](#5-sherpa-onnx-v1133-kotlin-api-规范补强-2) | PoC-A 真机验证 sherpa-onnx Direct Boot + onnxruntime CE-only API 加载链路 | Phase 1 | Phase 1 启动前（PoC-A 硬关卡） |
| 6 | R2 | [03 §5.2](./03_architecture_detail.md) | sherpa-onnx Kotlin API 调用样例待补（参考官方 SherpaOnnxVadAsr） | Phase 1-2 | Phase 1-2 实施时 |
| 7 | R3 | [15 §8](#8-遗留待补项建议级对应-phase-实施时补)（= 上表 #4） | APK 被 MIUI 杀后检测 + 复活通知流程（BootReceiver 检测 lastExitWasAbnormal） | Phase 4 | Phase 4 启动前 |
| 8 | R3 | [15 §6.4](#64-native-crash-处理第六轮-r3-新增) | native crash 1h 内重复 ≥3 次自动降级禁用对应能力策略 | Phase 4 | Phase 4 实施时 |
| 9 | R3 | [15 §6.5](#65-资源泄漏防护清单第六轮-r3-新增) | debug 构建资源泄漏检测断言（每次状态退出断言资源已释放） | Phase 1 | Phase 1 实施时 |
| 10 | R3 | [03 §5.2](./03_architecture_detail.md) | 错误重试的具体退避策略待补 | Phase 1-2 | Phase 1-2 实施时 |
| 11 | R4 | [15 §8](#8-遗留待补项建议级对应-phase-实施时补)（= 上表 #1） | 1500 行 JNI wrapper 工程量分解表（拆 8 个分项） | Phase 1 | Phase 1 启动前 |
| 12 | R4 | [15 §8](#8-遗留待补项建议级对应-phase-实施时补)（= 上表 #3） | Phase 6/7/9 子故障错误码细分表（4 工具 4xx/5xx/timeout + GLM-4V） | Phase 6/7/9 | 各 Phase 启动前 |
| 13 | R4 | [03 §6.1](./03_architecture_detail.md) | Phase 7-10 内存门槛"仅条件启用"强制条件（ScreenCapture/OCR/Cap1/switchProfile 须先 unloadModel） | Phase 7-10 | 各 Phase 启动前 |
| 14 | R5 | [DECISIONS.md D14](../DECISIONS.md) | Phantom Mic 上游活跃度复查（第四轮已确认停滞，复查转为降级方案可用性） | Phase 1 | Phase 1 启动前 |
| 15 | R5 | [14 Stage 1 PoC-B](./14_feasibility_recheck_and_plan.md) | WhatsMicFix-LSPosed 改造 PoC-B 真机验证（scope 改造 + 锁屏录音 ≥80%） | Phase 1 | Phase 1 启动前（PoC-B 硬关卡） |
| 16 | R5 | [DECISIONS.md D9](../DECISIONS.md) | LSPosed Vector v2.0 后续更高版本复查（以实际 release 页面为准） | Phase 1 | Phase 1 启动前 |
| 17 | R5 | [15 §8](#8-遗留待补项建议级对应-phase-实施时补)（= 上表 #2） | CI release workflow 蓝图文件（`.github/workflows/release.yml` + keystore 流程） | Phase 5 | Phase 5 启动前 |
| 18 | R6 | [DECISIONS.md D22](../DECISIONS.md) | TTS vits-zh-ll / matcha-baker（Data-Baker NC）许可待 Phase 5 发布前解决 | Phase 5 | Phase 5 发布前 |
| 19 | R6 | [DECISIONS.md D23](../DECISIONS.md) | KWS WenetSpeech 训练数据许可矛盾待 Phase 5 发布前确认 | Phase 5 | Phase 5 发布前 |

**维度分布**：R1=3 / R2=3 / R3=4 / R4=3 / R5=4 / R6=2 = 19 项。**阻塞 Phase 1 启动的硬关卡**：#3 / #5 / #14 / #15 / #16（共 5 项 Phase 1 启动前必做）；其余 14 项为各 Phase 实施时补。

**Phase 分布**：Phase 1 启动前 7 项 / Phase 1-2 实施时 3 项 / Phase 4 2 项 / Phase 5 3 项 / Phase 6-10 2 项 / Phase 11+ 2 项。

---

## 9. 引用关系

本文档被以下文件引用（2026-06-30 第六轮 R3/R4 补强后实际状态）：

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
