# Kotlin

> **定位**：相对 Java 的补充笔记——**语言特性 + 协程**；Android / JVM API 与 Spring 仍大量等同 Java 心智模型。Android 总览见 [Android.md](../Android/Android.md)。

---

## 与 Java 互操作

- **可为空性**：平台类型（Java）、`@Nullable`/`@NonNull` 注解推断。
- **静态成员**：`@JvmStatic`、`@JvmOverloads`、`@JvmName`。
- **文件级函数**：编译到 `FilenameKt` 类。
- **SAM**：Java 单抽象接口处 lambda 自动转换；Kotlin 函数类型对接时注意装箱。
- **异常**：Java 受检异常在 Kotlin **不经编译器强制**。

---

## 空安全

- **`T` vs `T?`**：`?.`、`?:`、`!!`、`let`、安全转换 `as?`。
- ** lateinit**：非 null；`::lateinit.isInitialized`（部分场景）。
- **可空与集合**：`List<Int?>` vs `List<Int>?`。

---

## 类型与语法糖

- **data class**：`copy`、解构；**避免**在领域模型中滥用含继承复杂图。
- **sealed class / interface**：穷举 when 分支（exhaustive when 随版本演化）。
- **object**、**companion object**、**内联类 `value class`**：性能与 API 表意。
- **类型别名** `typealias`、**范围** `1..10`、**中缀** `infix`。
- **委托**：`by`、**类委托**、**属性委托**（`lazy`、`observable`、自定义）。

---

## 函数式特性

- **高阶函数与 lambda**；**带接收者的 Lambda**（DSL：`apply`、`with`、`run`、`also`、`let`）。
- **内联函数 `inline`**：`reified` 泛型实化。
- **序列 `Sequence`**：惰性链式 vs **Iterable** 及时求值。

---

## 泛型

- **声明点变型** `out`/`in`；**类型投影** `Foo<out Bar>`。
- **星投影** `*`。
- 与 Java 通配符 **`PECS`** 对照记忆。

---

## 协程 Coroutines（重点）

**动机**：回调地狱、`Executor` 编排；**结构化并发**（父 Job 取消子任务）。

**核心概念**

- **`suspend`**：挂起点；不占用线程（依调度器）。
- **CoroutineContext**：`Job` + `Dispatcher` + 用户元素。
- **Dispatchers**：`Main`（Android 主）、`Default`（CPU）、`IO`（阻塞 IO）、`Unconfined`（慎用）。
- **CoroutineScope**：生命周期绑定 **`SupervisorJob`** vs **Job**（失败传播差异）。
- **`launch` vs `async`**：`Deferred.await()`。
- **异常**：`CoroutineExceptionHandler`；**`SupervisorJob`** 子协程失败隔离。

**Flow**

- **冷流**：类似 Reactive Streams；`collect`、`flatMapMerge`、`stateIn`、`shareIn`。
- **Channel**：多生产者消费者；**背压**与 `BUFFERED` 策略。
- **冷 / 热**：StateFlow / SharedFlow **热**缓存。

**Android 衔接**

- **`lifecycleScope`**：`Lifecycle` 绑定销毁。
- **`viewModelScope`**：ViewModel **clear** 取消。
- **`repeatOnLifecycle`**：`collect` 安全生命周期。
- **避免**：在 **GlobalScope** 长期任务（泄漏与取消难）。

---

## 并发补充（JVM）

- **`@Volatile`**：可见性；非复合原子性。
- **`Mutex`、`Semaphore`**（协程）：替代部分 `synchronized` 场景。
- **线程池**：协程底层仍映射线程；阻塞调用用 **`withContext(Dispatchers.IO)`**。

---

## 标准库精选

- **`Result`**、`runCatching`。
- **文件 IO**：`kotlin.io`、`okio`（若引入）。
- **时间**：`kotlin.time`、`java.time`（Java 8+ API）。

---

## 附录：版本特性自查

- **K2 编译器**、**上下文接收者**（预览/演进特性按官方 Release Notes 维护一页链接）。
