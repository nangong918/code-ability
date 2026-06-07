# JNI调用SDK大数据返回OOM问题解决方案文档

## 1 问题概述

项目中通过JNI调用底层SDK获取设备JSON数据，小数据量场景下功能正常。但在高负载测试场景（4w张接收卡批量数据查询）时，APK频繁发生OOM崩溃。初始通过配置`android:largeHeap="true"`增大应用堆内存上限，问题未得到解决，严重影响高并发、大数据量设备查询业务的稳定性。

## 2 原有方案及核心问题分析

### 2.1 原有实现方案

原有JNI交互逻辑为：底层SDK直接返回`std::string`完整数据，JNI层将完整字符串一次性转换为`jstring`对象，直接返回给Java层，Java层直接接收完整大字符串进行业务处理。整体为**一次性全量内存拷贝、整块大数据传输**模式。

### 2.2 核心问题根源

#### 2.2.1 堆内存碎片导致大内存申请失败（核心原因）

`android:largeHeap="true"`仅提升应用整体堆内存上限，无法解决**内存碎片化**问题。高负载场景下，应用频繁创建、销毁对象，Java堆内存被分割为大量零散小内存块，系统总剩余内存充足，但无法找到一块连续的百MB级内存块用于存储完整的设备JSON数据，最终触发OOM。

#### 2.2.2 双重内存拷贝，瞬时内存峰值过高

原有流程存在双倍完整数据内存占用：Native层SDK生成的`std::string`占用一份Native堆内存，JNI转换为`jstring`时，会将数据完整拷贝至Java堆，生成第二份完整数据。4w张卡数据量级可达百MB级，瞬时双重内存峰值直接压垮虚拟机内存阈值。

#### 2.2.3 JString编码扩容，加剧内存压力

Java层`jstring`基于UTF-16编码存储，而SDK返回的JSON数据为UTF-8编码。JSON中大量符号、中文会出现1字节转2字节的编码扩容，百MB级原始数据会翻倍占用Java堆内存，大幅提升OOM概率。

#### 2.2.4 大对象常驻堆内存，GC回收不及时

一次性返回的超大字符串属于Java大对象，虚拟机GC对大对象的回收效率极低，高负载连续请求下，大量大对象堆积在堆内存，无法及时释放，最终导致内存溢出。

## 3 整体解决方案

摒弃原有**一次性全量返回大字符串**的交互模式，采用**Native分块流式传输+Java边解码边输出**的轻量化方案，从根源解决大内存连续申请、双倍拷贝、内存峰值过高问题。

### 3.1 核心优化思路

1. **内存阵地转移**：将大数据缓冲区从Java堆转移至Native堆（malloc申请），Native堆内存连续性更好、碎片化影响极低，可稳定支撑百MB级数据存储。

2. **分块流式传输**：固定256KB小块缓冲区循环复用，替代整块大数组传输，彻底规避Java堆连续大内存申请需求，抹平内存碎片影响。

3. **消除双重内存拷贝**：Native层仅保留一份原始数据，通过分块直接写入Java输出流，不再创建全局大`jstring`，杜绝双倍内存占用。

4. **流式解码无大对象**：Java层自定义流式解码输出工具，边接收字节、边解码、边写入业务Writer，全程不存储完整大数据字符串，Java堆内存峰值稳定可控。

5. **完善资源防护**：增加完整的JNI局部引用释放、内存手动释放、虚拟机异常检测逻辑，避免高负载下内存泄漏、虚拟机异常崩溃。

### 3.2 整体业务流程

SDK生成UTF-8原始数据（Native堆）→ JNI分块256KB读取数据 → 循环写入Java OutputStream → Java层流式UTF-8解码 → 逐段写入业务Writer完成响应，全程无完整大对象、无大块连续内存申请。

## 4 核心代码实现摘录

### 4.1 JNI Native层核心代码（分块流式推送）

```cpp
JNIEXPORT jint JNICALL
Java_com_color_cltdevice_CLTDeviceJniHelper_CLTReceiverJsonInfoSpecial(JNIEnv *env, jobject,
                                                                       jobject dataObj) {
    if (!SearchDevice()) {
        return T9E_NOT_FIND_ANY_DEVICE;
    }

    jclass dataClass = env->GetObjectClass(dataObj);
    if (nullptr == dataClass) {
        return T9_E_FAIL;
    }

    // 获取Java层方法、字段ID
    jmethodID mId_getParam = env->GetMethodID(dataClass, "getParam", "()[B");
    jfieldID fidOut = env->GetFieldID(dataClass, "mOutputStream", "Ljava/io/OutputStream;");
    jfieldID fidCap = env->GetFieldID(dataClass, "mNativeBufferMaxBytes", "I");
    if (nullptr == mId_getParam || nullptr == fidOut || nullptr == fidCap) {
        env->DeleteLocalRef(dataClass);
        return T9E_PTR_NULL;
    }

    jobject outputStream = env->GetObjectField(dataObj, fidOut);
    if (nullptr == outputStream) {
        env->DeleteLocalRef(dataClass);
        return T9E_PTR_NULL;
    }

    // 初始化Native缓冲区容量，最大限制512MB，防止内存溢出
    jint capacityJ = env->GetIntField(dataObj, fidCap);
    int capacity = (capacityJ > 0) ? static_cast<int>(capacityJ) : (128 * 1024 * 1024);
    const int kMaxNativeBuffer = 512 * 1024 * 1024;
    if (capacity > kMaxNativeBuffer) {
        capacity = kMaxNativeBuffer;
    }

    // 获取请求参数
    jbyteArray requestArr = static_cast<jbyteArray>(env->CallObjectMethod(dataObj, mId_getParam));
    if (nullptr == requestArr) {
        env->DeleteLocalRef(outputStream);
        env->DeleteLocalRef(dataClass);
        return T9E_PTR_NULL;
    }

    int requestLength = env->GetArrayLength(requestArr);
    char *pRequestStr = reinterpret_cast<char *>(env->GetByteArrayElements(requestArr, nullptr));

    // Native堆申请缓冲区，规避Java堆碎片问题
    char *pResultStr = static_cast<char *>(std::malloc(static_cast<size_t>(capacity)));
    if (nullptr == pResultStr) {
        env->ReleaseByteArrayElements(requestArr, reinterpret_cast<jbyte *>(pRequestStr), 0);
        env->DeleteLocalRef(requestArr);
        env->DeleteLocalRef(outputStream);
        env->DeleteLocalRef(dataClass);
        return T9E_MALLOC_ERROR;
    }

    // 调用SDK获取原始UTF-8数据
    int resultLen = capacity;
    syslog(LOG_INFO, "CLTReceiverJsonInfoSpecial::JNI::成功malloc；resultLen = %d", resultLen);
    CLTReceiverJsonCommand(pRequestStr, requestLength, pResultStr, &resultLen);

    // 释放请求参数资源
    env->ReleaseByteArrayElements(requestArr, reinterpret_cast<jbyte *>(pRequestStr), 0);
    env->DeleteLocalRef(requestArr);

    // 获取OutputStream写入、刷新方法
    jclass osClass = env->GetObjectClass(outputStream);
    jmethodID midWrite = env->GetMethodID(osClass, "write", "([BII)V");
    jmethodID midFlush = env->GetMethodID(osClass, "flush", "()V");
    env->DeleteLocalRef(osClass);
    if (nullptr == midWrite) {
        std::free(pResultStr);
        env->DeleteLocalRef(outputStream);
        env->DeleteLocalRef(dataClass);
        return T9E_PTR_NULL;
    }

    // 固定256KB分块缓冲区，循环复用，控制内存峰值
    static const jsize kStreamChunkBytes = 256 * 1024;
    jbyteArray chunkArr = env->NewByteArray(kStreamChunkBytes);
    if (nullptr == chunkArr) {
        std::free(pResultStr);
        env->DeleteLocalRef(outputStream);
        env->DeleteLocalRef(dataClass);
        return T9E_MALLOC_ERROR;
    }

    // 分块流式写入Java OutputStream
    const int chunkInt = static_cast<int>(kStreamChunkBytes);
    const int totalRoundsExpected = (resultLen <= 0) ? 0 : (resultLen + chunkInt - 1) / chunkInt;
    syslog(LOG_INFO,
           "CLTReceiverJsonInfoSpecial::JNI::开始拷贝到OutputStream resultLen=%d chunkBytes=%d 预计轮次=%d",
           resultLen, chunkInt, totalRoundsExpected);

    bool streamOk = true;
    int round = 0;
    int offset = 0;
    for (; offset < resultLen;) {
        int n = resultLen - offset;
        if (n > chunkInt) {
            n = chunkInt;
        }
        // 单块数据拷贝
        env->SetByteArrayRegion(chunkArr, 0, n, reinterpret_cast<const jbyte *>(pResultStr + offset));
        if (env->ExceptionCheck()) {
            env->ExceptionClear();
            streamOk = false;
            break;
        }
        // 写入Java流
        env->CallVoidMethod(outputStream, midWrite, chunkArr, 0, n);
        if (env->ExceptionCheck()) {
            env->ExceptionClear();
            streamOk = false;
            break;
        }
        offset += n;
        ++round;
        // 定时打印进度日志
        if (round % 30 == 0) {
            syslog(LOG_INFO,
                   "CLTReceiverJsonInfoSpecial::JNI::拷贝进度 第%d轮 已拷贝%d字节/目标%d字节",
                   round, offset, resultLen);
        }
    }

    // 刷新缓冲区，完成数据推送
    if (streamOk && nullptr != midFlush) {
        env->CallVoidMethod(outputStream, midFlush);
        if (env->ExceptionCheck()) {
            env->ExceptionClear();
            streamOk = false;
        }
    }

    syslog(LOG_INFO,
           "CLTReceiverJsonInfoSpecial::JNI::拷贝与flush结束 streamOk=%d 实际轮次=%d 已写入字节=%d/目标%d字节",
           streamOk ? 1 : 0, round, offset, resultLen);

    // 统一释放所有资源，杜绝内存泄漏
    std::free(pResultStr);
    env->DeleteLocalRef(chunkArr);
    env->DeleteLocalRef(outputStream);
    env->DeleteLocalRef(dataClass);

    return streamOk ? static_cast<jint>(resultLen) : static_cast<jint>(T9_E_FAIL);
}
```

### 4.2 Java层流式解码工具类核心代码

自定义`Utf8BytesToWriter`，适配HTTP响应Writer场景，实现字节流实时解码输出，全程无完整大对象缓存：

```java
/**
 * JNI 按固定块写入 UTF-8 字节；
 * 适配HttpServletResponse场景，已获取Writer时，禁止重复获取OutputStream
 * 流式解码写入，避免ByteArrayOutputStream扩容双倍堆分配导致OOM
 */
private static final class Utf8BytesToWriter extends OutputStream {
    /** JNI单块传输256KB，预留冗余字节 */
    private static final int BUF_CAP = 260 * 1024;
    /** 首尾采样日志长度，用于问题排查，不缓存完整数据 */
    private static final int SAMPLE_CHARS = 256;

    private final Writer writer;
    private final CharsetDecoder decoder = StandardCharsets.UTF_8.newDecoder()
            .onMalformedInput(CodingErrorAction.REPLACE)
            .onUnmappableCharacter(CodingErrorAction.REPLACE);
    private final ByteBuffer buf = ByteBuffer.allocate(BUF_CAP);
    // 首尾采样缓存，用于日志排查
    private final StringBuilder firstSample = new StringBuilder(SAMPLE_CHARS);
    private final char[] tailRing = new char[SAMPLE_CHARS];
    private int tailWritePos = 0;
    private int totalChars = 0;

    Utf8BytesToWriter(Writer writer) {
        this.writer = writer;
    }

    @Override
    public void write(byte[] b, int off, int len) throws IOException {
        // 循环分块解码，避免单次处理数据过大
        while (len > 0) {
            int n = Math.min(len, buf.remaining());
            buf.put(b, off, n);
            off += n;
            len -= n;
            buf.flip();
            decodeBlocks(false);
            buf.compact();
        }
    }

    @Override
    public void write(int b) throws IOException {
        write(new byte[]{(byte) b}, 0, 1);
    }

    /**
     * 分块解码UTF-8字节流，实时写入Writer
     */
    private void decodeBlocks(boolean endOfInput) throws IOException {
        while (true) {
            CharBuffer out = CharBuffer.allocate(16384);
            CoderResult cr = decoder.decode(buf, out, endOfInput);
            out.flip();
            if (out.hasRemaining()) {
                String chunk = out.toString();
                writer.append(chunk);
                captureSamples(chunk);
            }
            // 数据读取完毕，退出循环
            if (cr.isUnderflow()) {
                break;
            }
            if (!cr.isOverflow()) {
                break;
            }
        }
    }

    /**
     * 结束流，处理跨块UTF-8码点、收尾解码
     */
    void finish() throws IOException {
        buf.flip();
        decodeBlocks(true);
        buf.compact();
        // 刷新解码器残留数据
        for (;;) {
            CharBuffer out = CharBuffer.allocate(64);
            CoderResult cr = decoder.flush(out);
            out.flip();
            if (out.hasRemaining()) {
                writer.append(out);
            }
            if (!cr.isOverflow()) {
                break;
            }
        }
    }

    /**
     * 采样数据首尾片段，用于日志排查，不缓存完整数据
     */
    private void captureSamples(String chunk) {
        int toHead = SAMPLE_CHARS - firstSample.length();
        if (toHead > 0) {
            firstSample.append(chunk, 0, Math.min(toHead, chunk.length()));
        }

        // 环形缓存尾部片段
        for (int i = 0; i < chunk.length(); i++) {
            tailRing[tailWritePos] = chunk.charAt(i);
            tailWritePos = (tailWritePos + 1) % SAMPLE_CHARS;
            totalChars++;
        }
    }

    // 日志采样获取方法
    String getFirstSample() {
        return firstSample.toString();
    }

    String getLastSample() {
        int count = Math.min(totalChars, SAMPLE_CHARS);
        if (count <= 0) {
            return "";
        }
        StringBuilder sb = new StringBuilder(count);
        int start = (totalChars >= SAMPLE_CHARS) ? tailWritePos : 0;
        for (int i = 0; i < count; i++) {
            sb.append(tailRing[(start + i) % SAMPLE_CHARS]);
        }
        return sb.toString();
    }

    @Override
    public void flush() throws IOException {
        writer.flush();
    }
}
```

## 5 方案优化亮点总结

1. **彻底解决内存碎片问题**：大数据存储迁移至Native堆，流式小块传输规避Java堆连续大内存申请，彻底摆脱`largeHeap`局限性。

2. **内存占用极致优化**：全程无双重数据拷贝、无完整大字符串缓存，Java堆内存峰值稳定在256KB级别，高负载4w+设备数据无OOM。

3. **编码效率提升**：直接传输原始UTF-8字节流，规避JString UTF-16编码扩容问题，减少50%左右内存冗余占用。

4. **高负载稳定性强**：完善的JNI资源释放、异常捕获、内存回收逻辑，杜绝内存泄漏、虚拟机异常崩溃，适配高并发批量查询场景。

5. **可观测性完善**：增加传输进度日志、数据首尾采样日志，方便线上问题排查，同时不占用大量内存存储日志数据。

## 6 落地注意事项

1. Native层所有malloc内存、JNI局部引用、字节数组必须手动释放，禁止遗漏，避免高负载下内存累积泄漏。

2. JNI虚拟机异常校验后必须主动清除异常，防止单次异常导致后续所有JNI调用失效。

3. 分块大小固定为256KB为最优区间，兼顾传输效率和内存稳定性，不建议随意修改。

4. 最大Native缓冲区限制512MB，防止极端场景下Native内存溢出，可根据业务上限微调。

5. Java层必须调用`finish()`方法完成流收尾解码，避免跨块UTF-8码点丢失、数据残缺问题。
