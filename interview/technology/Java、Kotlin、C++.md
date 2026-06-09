# Java、Kotlin、C++


## Java


## Kotlin

### 协程 vs 线程 / 协程是什么 /suspend 是什么

* 线程是 OS 层面的，由内核调度，重量级、开销大、切换成本高；
* 协程是语言 / 库层面的，由用户态调度，轻量级、开销极小
  - 协程就是 可挂起、可恢复的轻量级执行单元；

Java 启动 10 万个线程（直接 OOM）
```java
// Java 线程：重量级，几千个就崩溃
public static void main(String[] args) {
    for (int i=0; i<100000; i++) {
        new Thread(() -> {
            try { Thread.sleep(1000); } 
            catch (Exception e) {}
        }).start();
    }
}
```
Kotlin 协程 启动 10 万个协程（轻松运行）
```kotlin
// 协程：极轻量，千万个都不崩
fun main() = runBlocking {
    repeat(100_000) {
        launch { // 开启 10 万个协程
            delay(1000)
        }
    }
}
```

* 什么是 suspend 挂起函数？

suspend 是挂起函数的修饰符，只能在协程 / 其他挂起函数中调用；
挂起 = 不阻塞线程，线程被释放；

Java可以通过Runnable实现类似的功能，但是：
Runnable 只能执行完，不能中途暂停；协程可以中途暂停，之后再回来继续执行。
```java
Runnable r = new Runnable() {
    public void run() {
        // 一旦开始，必须跑完
        // 不能中途暂停
        task1();
        task2();
        task3();
    }
};
```

协程 suspend 函数（可暂停 + 可恢复）
```kotlin
suspend fun task() {
    task1()

    // 挂起！线程去干别的！
    delay(1000)

    // 恢复！继续执行
    task2()
}
```

### withContext / 异常处理 / Flow / 延迟初始化

* withContext () 用途？
  - 切换协程的调度器（线程）；
  - 必返回结果，是挂起函数；
  - 常用：
    - Dispatchers.IO → 网络、数据库
    - Dispatchers.Main → 主线程（Android）
    - Dispatchers.Default → CPU 密集、计算、FFmpeg 软解码、算法
    - Unconfined → 几乎不用

```kotlin
// IO密集型
suspend fun fetchData(): String {
    // 切换到 IO 线程
    return withContext(Dispatchers.IO) {
        // 网络请求 / 数据库
        "result"
    }
}
// CPU密集型
suspend fun decoder(): Boolean {
    return withContext(Dispatchers.Default) {
        // FFmpeg 软解码（纯计算）返回解码成功结果
        ffmpeg.decode(h264Data)
    }
}
```

* 协程如何处理异常？

  - 结构化异常处理：
    - try-catch 直接捕获
    - CoroutineExceptionHandler 全局捕获
    - launch 异常自动向上传播

  - async 必须在 await 时捕获
  - 父协程异常 → 子协程全部取消
  - 子协程异常 → 父协程取消

````kotlin
// try-catch 捕获
launch {
    try {
        fetchData()
    } catch (e: Exception) {
        // 处理异常, 协程外部无法处理
    }
}

// 全局异常捕获器：文件存储失败统一处理
val saveFileHandler = CoroutineExceptionHandler { coroutineContext, throwable ->
    // 异常处理：权限不足、磁盘满、路径错误
    Log.e("File", "保存失败：${throwable.message}")
    Toast.makeText(context, "聊天记录保存失败", Toast.LENGTH_SHORT).show()
}

// 协程：保存 SSE 收到的AI回复到文件
lifecycleScope.launch(Dispatchers.IO + saveFileHandler) {
    val content = aiTextView.text.toString()
    saveChatRecordToFile(content) // 可能崩溃
}

// 挂起函数：保存文件
suspend fun saveChatRecordToFile(content: String) {
    // 模拟异常：权限不足
    throw IOException("SD卡不可写，无权限")
}

// launch 异常自动向上传播
launch(handler) { }
````

### Kotlin 协程中的 Flow 是什么？

Flow 是协程的异步数据流，冷流、轻量、安全，专门处理连续异步数据。

eg: 处理AI的SSE流式回复；跟Flutter的Stream很像

```kotlin
// 模拟：SSE 流式接收 AI 回复（一段一段发数据）
fun aiChatSSE(message: String): Flow<String> = flow {
    // 1. 开始请求
    emit("AI思考中...")

    // 模拟SSE流式返回：来一段发一段
    val aiReplyList = listOf(
        "你", "好", "，", "我", "是", "AI",
        "，", "很", "高", "兴", "为", "你", "服", "务！"
    )

    for (word in aiReplyList) {
        delay(200) // 模拟网络流延迟
        emit(word) // 🔥 一段一段发数据
    }
}

// 使用：一段一段收数据（打字机效果）
lifecycleScope.launch {
    aiChatSSE("你好！").collect { word ->
        // 🔥 一段一段收数据，实时更新UI
        aiTextView.append(word)
    }
}
```

### Kotlin与Java中伴生对象和静态成员有什么区别？

Java static 是 JVM 级静态；Kotlin 伴生对象是单例对象，功能更强，可继承可扩展，`@JvmStatic` 可兼容 Java 静态行为。

```kotlin
class Test{
    companion object { @JvmStatic val a = 1 }
}
```

