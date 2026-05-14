# Java

> **定位**：已有多年 Java 经验（含 **Spring Boot**、**Android** 等），本文只做 **知识架构梳理与笔记索引**，不写学习路线。  
> 章节合并为「大面」：语法一块收拢，框架一块收拢；需要细节时查官方文档与源码。

---

## 一、语言与语法总览（合并章）

> 日常几乎不靠「背语法」，此处作速查与查漏补缺索引。

- **词法与基础类型**：字面量、`var`、包装类型与缓存、`BigDecimal`/`BigInteger` 场景
- **运算符与控制流**：`switch` 表达式、`yield`、模式匹配（`instanceof`、switch、`record` 解构等，按所用版本）
- **方法与参数**：重载、可变参数、`this()`、`super()`、方法引用
- **类与接口**：嵌套类（静态 / 内部 / 局部 / 匿名）、`enum`、`record`、密封类 `sealed`
- **包与模块**：包访问、`module-info.java`（JPMS，按需）
- **异常体系**：受检 / 非受检、`try-with-resources`、`Throwable` 层次

---

## 二、面向对象、继承与多态（在 JVM 语义下）

- **抽象类与接口**：默认方法、静态方法、私有方法、`@FunctionalInterface`
- **继承**：单继承、`final`、`Object` 契约（`equals`/`hashCode`/`toString`/`clone`）
- **多态与绑定**：虚调用、桥方法（泛型擦除相关时的直觉）
- **设计表达**：何时用 `record`、何时用继承 / 组合（个人准则笔记）

---

## 三、泛型、反射、注解

- **泛型**：擦除、边界、通配符 `PECS`、类型推断
- **反射**：`Class`、`MethodHandle`（按需）、性能与安全边界
- **注解**：保留策略、可重复注解、处理器（APT）是否涉足
- **字节码与插桩**（可选）：Agent、常见监控切入点

---

## 四、集合框架 · 深度结合「数据结构 + 算法」

> 复习结构/复杂度时，强制对应 **JDK 容器** 与 **Stream / Collections** API。

**线性 / 顺序结构**

- 动态数组 ↔ `ArrayList`；双端队列 ↔ `ArrayDeque`、`LinkedList` 取舍
- 栈 / 队列语义 ↔ `Deque` 接口与各实现差异

**哈希**

- `HashMap` / `LinkedHashMap` / `ConcurrentHashMap`：负载、扩容、线程安全差异

**有序与树形语义**

- `TreeMap` / `TreeSet`；`ConcurrentSkipListMap`（按需）

**不可变与拷贝**

- `List.of`、`Map.of`、`copyOf`、防御性拷贝习惯

**并发集合**

- `BlockingQueue` 族、`CopyOnWriteArrayList` 适用场景

**算法与 Stream**

- `Collections.sort` / `List.sort`、`Arrays.parallel*`；Stream：惰性、中间 / 终止操作、并行流代价

**实务**

- 时间复杂度笔记与选型表（自建）；避免「默认 `HashMap` 一切」的例外情形

---

## 五、I/O、NIO、序列化

- **经典 IO**：`Reader`/`Writer`、`InputStream`/`OutputStream`、装饰器模式痕迹
- **NIO / NIO.2**：`Buffer`、`Channel`、`Selector`（概念）；`Path`、`Files`、`WatchService`
- **序列化**：Java 原生序列化利弊、`Serializable`、版本 `serialVersionUID`；JSON（Jackson 等）与 Protobuf 等对接 Spring/Android 时单独记

---

## 六、并发与 Java 内存模型（JMM）

- **线程基础**：`Thread`、`Runnable`、`Executor` 体系、`Future`/`CompletableFuture`
- **同步**：`synchronized`、`volatile`、`java.util.concurrent.locks`
- **工具类**：`CountDownLatch`、`Semaphore`、`CyclicBarrier`、线程池参数与拒绝策略
- **JMM**：happens-before、可见性与有序性（与排查 Bug 相关的笔记）
- **虚拟线程**（Java 21+）：适用场景与不要阻塞 carrier 的直觉
- **Android 补充**：主线程 / `Handler` / `Looper`、与 JVM 并发模型的差异（若笔记跨端）

---

## 七、JVM、性能与排错

- **类加载**：双亲委派、自定义 ClassLoader 场景
- **运行时数据区**：栈 / 堆 / 方法区（元空间）、直接内存
- **GC**：分代与主流收集器名字、GC 日志与 STW 直觉
- **JIT**：热点、内联、逃逸分析（概念）
- **调优入口**：堆参数、GC 选择、分析工具（`jcmd`、`jstack`、MAT、async-profiler 等按需列清单）
- **JNI 边界**：本地栈、Critical Section、与 Native 交互时的崩溃排查线索（与 C++ 笔记交叉引用）

---

## 八、构建与依赖：Maven、Gradle

**Maven**

- 生命周期、`dependency`/`plugin`、多模块、`BOM`、与 Spring 依赖管理

**Gradle**

- Kotlin DSL、任务图、`implementation` vs `api`、与 Android Gradle Plugin 关系

**共通**

- 仓库与镜像、版本冲突解析、可重复构建

---

## 九、Spring 与 Spring Boot 生态（合并大章）

> 按项目用到的组件挂索引，避免重复造「子教程」。

- **核心**：IoC 容器、`@Configuration`、`Bean` 作用域、条件装配
- **AOP**：代理方式、切点、与事务边界
- **Web**：`Spring MVC` / WebFlux 选型、参数绑定、过滤器与拦截器
- **数据**：`JdbcTemplate`、JPA / MyBatis 等持久层抽象；事务传播与隔离
- **安全**：Spring Security 过滤器链、认证 / 授权概念位
- **整合**：Validation、Scheduling、Cache、`spring-boot-actuator`、配置（profile、配置树）
- **云与分布式**（按需）：Spring Cloud 组件名片（注册发现、配置、网关、熔断）

---

## 十、Android（Java / Kotlin 混合视角）

> 你已熟悉，此处作「模块地图」与跨语言笔记挂钩。

- **组件**：Activity / Service / BroadcastReceiver / ContentProvider（生命周期与入口）
- **构建**：Gradle 变体、ABI、`CMake`/NDK 与 JNI 衔接
- **Jetpack**（按需列）：Lifecycle、ViewModel、Navigation、Room、WorkManager…
- **进程与线程**：主线程约束、`Executor`、与 Native 音频视频线程协作
- **系统**：权限模型、前后台、进程优先级（流媒体后台场景）

---

## 十一、网络与中间件对接（Java 侧）

- **HTTP 客户端**：`HttpClient`（11+）、OkHttp、RestTemplate / WebClient
- **Netty / Reactor**（若使用）：线程模型与背压直觉
- **消息 / 缓存**：JMS、Kafka/Rabbit 客户端、Redis 客户端在 Spring 中的常见装配（按需）

---

## 十二、测试、可观测性与工程质量

- **单元 / 集成测试**：JUnit 5、Mockito、Testcontainers（按需）
- **日志**：SLF4J + Logback / Log4j2、结构化日志与追踪 ID
- **指标与追踪**：Micrometer、OpenTelemetry、Spring Boot Actuator 端点
- **静态分析与风格**：Checkstyle、SpotBugs、Error Prone、Spotless（按需）

---

## 十三、语言演进与现代特性速查（按 JDK 版本自选）

- **模块系统**、**文本块**、**Records**、**密封类**、**Pattern Matching**、**Virtual Threads**、**Structured Concurrency**（预览特性单独标注）

---

## 附录（笔记占位）

- 常用依赖版本与 BOM 记录（Spring Boot / Android Gradle Plugin）
- 生产故障案例索引（OOM、死锁、GC、JNI）
- 与个人 **C++ / JNI** 笔记的交叉引用（流媒体、RK）
