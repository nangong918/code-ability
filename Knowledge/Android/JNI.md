# JNI

> Java/Kotlin 与 **C/C++** 的桥梁；Android NDK 开发、音视频编解码、图像处理、OpenSL/OpenMAX、厂商 SDK 等多依赖 JNI。C++ 通用语法与工程实践见 [C++.md](../Language/C++.md)。

---

## JNI 基本模型

- **JNI**：定义 C 调用约定与类型映射；Android 用 **ART** 加载 `.so`，通过 `System.loadLibrary` / `System.load`。
- **命名符号**：`Java_pack_Class_method`；或使用 **`RegisterNatives`** 显式绑定（推荐利于重构）。
- **ABI**：`armeabi-v7a`、`arm64-v8a`、`x86`、`x86_64`；Gradle `ndk { abiFilters }`；Fat APK vs App Bundle + 分包。

---

## JNIEnv 与线程

- **JNIEnv**：线程局部；**禁止跨线程直接使用** 其它线程缓存的 `JNIEnv*`。
- **附加**：Native 回调线程若未附加 JVM：`AttachCurrentThread` / `DetachCurrentThread`（配对）；**Attach** 后获取当前线程有效 `JNIEnv*`。
- **全局 VM**：`JavaVM*` 可保存；通过 `GetEnv` / `AttachCurrentThread` 取 `JNIEnv`。

---

## 引用类型：局部 / 全局 / 弱全局

- **局部引用**：Native 返回前多数自动释放；长时间 Native 循环中大量 New 对象需 **`DeleteLocalRef`** 防 **局部引用表溢出**。
- **全局引用**：`NewGlobalRef` / `DeleteGlobalRef`；跨线程、跨多次调用持有 Java 对象。
- **弱全局**：`NewWeakGlobalRef`；可能被 GC；使用前 **`NewLocalRef` 晋升** 或检查存活。

---

## 异常处理

- JNI 函数失败可能 **挂起 Java 异常**；后续 JNI 调用多为 **no-op** 直至 **`ExceptionCheck` / `ExceptionClear`**。
- **禁止**：在未 Clear 的情况下盲目继续 Native 逻辑。
- **抛出**：`ThrowNew`；Kotlin 异常亦可映射。
- **最佳实践**：Native 边界 **`try/catch` C++** + JNI 异常检查双保险。

---

## 字符串与数组

- **`GetStringUTFChars` / `ReleaseStringUTFChars`**：注意 UTF-8 与修改语义。
- **数组**：`GetByteArrayElements` / `ReleaseByteArrayElements`；`GetPrimitiveArrayCritical` **临界区尽量短**，避免阻塞或 JNI 重入。
- **拷贝 vs 钉扎**：模式 `isCopy`；实时路径避免不必要拷贝。

---

## Direct ByteBuffer

- **`NewDirectByteBuffer` / `GetDirectBufferAddress`**：与 Java `ByteBuffer.allocateDirect` 对应。
- **用途**：零拷贝友好、音视频缓存；仍要注意 **对齐、容量、生命周期**（Direct Buffer 受 GC 影响）。

---

## RegisterNatives

```c++
JNI_OnLoad(JavaVM* vm, void* reserved) {
    JNIEnv* env;
    vm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6);
    JNINativeMethod methods[] = { ... };
    jclass clazz = env->FindClass("com/example/NativeLib");
    env->RegisterNatives(clazz, methods, sizeof(methods)/sizeof(methods[0]));
    return JNI_VERSION_1_6;
}
```

- **优点**：方法名重构只改注册表；避免超长 C 符号名。
- **注意**：`FindClass` 在 **非 Java 调用栈**的线程上可能找不到应用 ClassLoader——可用 **缓存 `jclass` 全局引用** 或在 Java 侧传入 `Class`。

---

## Kotlin 与 JNI

- **名称修饰**：Kotlin 顶层函数、内部类、`object` 等方法名被编译器改写；需 **`javap -s`** 或用 **`@JvmName` / `@JvmStatic`** 固定导出符号。
- **推荐**：为 JNI 暴露 **明确的 `@JvmStatic external fun`** 包装层。

---

## NDK 构建

- **CMake**：主流；`externalNativeBuild { cmake { ... } }`。
- **ndk-build**：`Android.mk` / `Application.mk`。
- **STL**：`c++_shared` vs `c++_static`；与依赖 `.so` 共享冲突排查。
- **链接**：`LOCAL_LDLIBS` / `target_link_libraries` 包含 `log`、`android`、`OpenSLES` 等。

---

## 典型场景备忘

**音视频**：解码输出 **时间戳、线程模型**（回调线程 vs Java）、环形缓冲；背压与丢帧策略。

**图像**：`Bitmap` `lockPixels` / JNI `AndroidBitmap_*` API（版本差异）。

**OpenGL / Vulkan**：上下文线程亲和；EGL 与 Surface 生命周期。

**稳定性**：SIGSEGV 查 **野指针、数组越界、Use-After-Free**；结合 **`ndk-stack`** 符号化。

---

## 工具链

- **`addr2line` / `llvm-symbolizer`**：栈回溯。
- **ASan / HWASan**：CMake 里打开（调试构建）。
- **Android Studio Debugger**：Java + Native 联合调试。

---

## 文档交叉索引

- C++ 语言与 RAII： [C++.md](../Language/C++.md)
- Binder / 跨进程：若 Native Service 参与 IPC，见 [IPC.md](IPC.md)
- Framework / HAL：见 [AndroidFramework.md](../AndroidFramework/AndroidFramework.md)
