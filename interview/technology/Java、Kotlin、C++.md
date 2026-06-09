# Java、Kotlin、C++


## Java

### JVM

#### 描述一下Java创建栈变量和堆变量

栈变量：存在栈里，方法里的局部变量、基本类型、引用变量
堆变量：存在堆里，new 出来的对象、数组本身

#### StackOverFlow与OOM的区别？分别发生在什么时候，JVM栈中存储的是什么，堆存储的是什么？

StackOverflowError（栈溢出）：栈深度不够，方法调用层级太多（多出现在无线递归）
OutOfMemoryError（内存溢出）：堆 / 栈 / 方法区内存不够用，新对象分配不了内存

### Java基础

#### StringBuilder和StringBuffer的异同？

StringBuilder 快不安全，StringBuffer 慢但安全。

#### 什么是泛型？有什么作用？泛型擦除呢？

* 泛型：把类型当作参数传递，实现代码模板化

* 泛型擦除：
  * Java 泛型是伪泛型，编译阶段生效，编译后的字节码会抹去泛型信息，最终还原为原始类型（Raw Type）。
  * 目的：向前兼容 JDK5 之前无泛型的代码。
  * 带来的影响：
    * 不能用泛型类型创建实例：new T() 报错（编译后 T 变成 Object，语义不符）
    * 不能用 instanceof 判断泛型类型：obj instanceof List<String> 编译报错
    * 泛型数组受限：无法直接创建 T[] 数组
    * 重载冲突：test(List<String>) 和 test(List<Integer>) 编译报错，擦除后都是 test(List)

#### Java异常机制中，异常Exception与错误Error区别

* Error：JVM 系统级崩溃，程序处理不了，只能跑路
* Exception：代码逻辑问题，程序可以捕获、处理、修复

#### finally中的代码一定会执行吗？try里有return，finally还执行么？为什么 finally 里面绝对不能写 return？

* finally中的代码一定会执行吗？
  只有 3 种极端情况，finally 不会执行：（即便在try和catch中写return也会执行）
   - 在 try 之前就崩溃 / 返回（没进 try）
   - JVM 直接退出（System.exit(0)）
   - JVM 崩溃、断电、线程被强制杀死

* 为什么 finally 里面绝对不能写 return？

finally 里写 return，会覆盖 try /catch 里的 return，导致业务逻辑错乱、异常被吃掉！
```java
public static int test() {
    try {
        return 10;  // 本来要返回 10
    } finally {
        return 20;  // finally 强行 return，直接覆盖！
    }
}
```
结果返回：20


#### Java的泛型中super 和 extends 有什么区别？
extends 只读（取数据），super 可写（存数据）

```java
// extends
public void test_extends() {
    // 可以是 List<Animal>、List<Dog>、List<Cat>
    List<? extends Animal> list = new ArrayList<Dog>();

    // 读：可以！取出来是 Animal
    Animal a = list.get(0);

    // 写：不行！编译器报错
    list.add(new Dog());  // 报错
    list.add(new Animal()); // 报错
}

public void test_super() {
    // 可以是 List<Animal>、List<Object>
    List<? super Dog> list = new ArrayList<Animal>();

    // 写：可以！能加 Dog 及其子类
    list.add(new Dog());
    list.add(new SmallDog());

    // 读：只能用 Object 接收
    Object obj = list.get(0);
    Dog d = list.get(0); // 报错！
}
```

#### Java中有几种引用关系，它们的区别是什么？

* 强引用（Strong Reference）
* 软引用（Soft Reference）
* 弱引用（Weak Reference）: 只要发生 GC（无论内存是否充足），一定会回收弱引用关联的对象。
* 虚引用（Phantom Reference）: 和普通垃圾对象一致，GC 扫描到即可回收

```java
// 弱引用
WeakReference<Object> weakRef = new WeakReference<>(new Object());
Object obj = weakRef.get();

// 虚引用
ReferenceQueue<Object> queue = new ReferenceQueue<>();
PhantomReference<Object> ref = new PhantomReference<>(new Object(), queue);
```

#### Java IO 流了解吗？你的JNI的大JSON拷贝到Java层是怎么实现的？用什么接收JNI的数据？为什么这么做？

java.nio.DirectByteBuffer

```java
// 分配直接内存（你说的 OutBuffer）
ByteBuffer buffer = ByteBuffer.allocateDirect(1024 * 1024);

// 传给 JNI
nativeReadJson(buffer);
```

* 为什么要用 DirectByteBuffer？

  - 零拷贝 / 少拷贝（最重要）
  - 堆内存：JNI → Java 必须拷贝两次
  - DirectBuffer：JNI 直接访问内存地址，只拷贝一次


#### 讲讲OSS断点上传，断点续传，有用到Java IO吗？上传100G文件服务器内存只有16G是怎么处理避免OOM的？

##### OSS 断点上传、断点续传是什么？

断点续传 = 大文件分片上传 + 断点记录 + 失败恢复

- 把大文件切成固定大小的分片（一般 5MB、10MB）
- 每个分片独立上传
- 记录已上传成功的分片（checkpoint）
- 网络中断 / 程序重启后，跳过已上传分片，只传未完成的
- 全部传完后，OSS 服务端自动合并成完整文件

##### 有没有用到 Java IO？用到哪些？

主要用到：
* BufferedInputStream（缓冲字节流）
  - 本地文件分片读取
  - 高效、省内存
* NIO 的 ByteBuffer
  - 作为分片数据缓冲区
* FileChannel（可选）
  - 大文件随机读取

用 BIO 流式读取，用 NIO Buffer 做分片缓存，不一次性加载整个文件。

##### 上传100G文件客户端和服务器内存只有16G是怎么处理避免OOM的？

* 客户端
  - 不加载整个文件到内存，使用 Java IO 流式读取
  - 文件切成 固定大小分片（5MB/10MB）
  - 每次只读取 一个分片到内存，上传完成立即释放
  - 内存占用 永远只有一个分片大小，与文件大小无关
  - 用到：`BufferedInputStream`、`ByteBuffer`

* 服务端
  - 不把整个文件缓存到内存
  - 使用 流式接收，收到一点数据就立即写入磁盘 / OSS，不堆积
  - 服务端内存只保留 少量接收缓冲区
  - 绝对不使用字节数组 / 内存队列缓存整个大文件
  - 分片独立存储，最后合并索引，不移动真实数据
  - 用到：OutputStream、Socket 输入流、NIO Channel

客户端代码
```java
import java.io.BufferedInputStream;
import java.io.File;
import java.io.FileInputStream;
import java.nio.ByteBuffer;

public class OssUploadClient {
    public void uploadBigFile(File file) throws Exception {
        // 分片大小 10MB（关键：固定小内存）
        int partSize = 10 * 1024 * 1024;
        long fileLength = file.length();

        // 流式读取文件（不会加载整个文件）
        try (FileInputStream fis = new FileInputStream(file);
             BufferedInputStream bis = new BufferedInputStream(fis)) {

            byte[] partBytes = new byte[partSize];
            ByteBuffer buffer = ByteBuffer.wrap(partBytes);

            int readLen;
            int partNumber = 1;

            // 循环读分片（内存永远只占 10MB）
            while ((readLen = bis.read(partBytes)) != -1) {
                buffer.limit(readLen);
                // 上传当前分片到服务端/OSS
                uploadPart(partNumber, buffer);
                partNumber++;
            }
        }
    }

    private void uploadPart(int partNumber, ByteBuffer buffer) {
        // 发送分片：网络输出流写入
        // 内存只占用一个分片，用完即释放
    }
}
```

服务端：流式接收（不会 OOM）
```java
import java.io.InputStream;
import java.io.OutputStream;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

public class OssUploadServer {
    public void receiveBigFile(InputStream socketInput) throws Exception {
        int bufferSize = 10 * 1024 * 1024;
        byte[] buf = new byte[bufferSize];
        int len;

        // 从 Socket 流里边读边写（不缓存全文件）
        while ((len = socketInput.read(buf)) != -1) {
            // 直接写入磁盘/OSS，不存内存
            writeToDiskOrOss(buf, len);
        }
    }

    // NIO 版本（零拷贝，更高效）
    public void receiveWithNIO(InputStream input) throws Exception {
        FileChannel channel = FileChannel.open(/* 输出文件路径 */);
        ByteBuffer buffer = ByteBuffer.allocateDirect(10 * 1024 * 1024);

        while (input.read(buffer.array()) != -1) {
            channel.write(buffer);
            buffer.clear();
        }
    }

    private void writeToDiskOrOss(byte[] data, int len) {
        // 落盘 / 写入 OSS，不占用堆内存
    }
}
```

Android下载BIO
```java
// Android 断点下载 / 大文件下载 标准代码
public void downloadFile(InputStream inputStream, OutputStream outputStream) throws Exception {
    // 固定小缓冲区，永远只占 8KB ~ 10MB 内存
    byte[] buffer = new byte[8192]; 
    int length;

    // 边下载 边写入文件，绝不放内存
    // inputStream.read () 会阻塞; 没数据时，线程卡住不动，直到网络来数据
    while ((length = inputStream.read(buffer)) != -1) {
        outputStream.write(buffer, 0, length);
    }

    inputStream.close();
    outputStream.close();
}
```

Android下载NIO
```java
public void downloadWithNIO(InputStream input, OutputStream output) throws Exception {
    // 用 Channel + Buffer, 支持非阻塞、多路复用
    FileChannel inChannel = ((FileInputStream) input).getChannel();
    FileChannel outChannel = ((FileOutputStream) output).getChannel();

    ByteBuffer buffer = ByteBuffer.allocateDirect(8192);

    while (inChannel.read(buffer) != -1) {
        buffer.flip();
        outChannel.write(buffer);
        buffer.clear();
    }
}
```

#### BIO、NIO 和 AIO 的区别？

AIO: 废弃
BIO（Blocking IO）（阻塞 IO）：一个连接一个线程，等数据时线程死等、阻塞，不做别的事。
NIO（Non-blocking IO）（非阻塞 IO）：一个线程管理多个连接，不等数据，有数据才处理。


`inputStream.read(buffer)`会阻塞，`inChannel.read(buffer)`不会阻塞






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

