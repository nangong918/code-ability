# C++

> **读者画像**：已掌握 **C**，具备 **部分 C++**；学习目标侧重 **Android JNI（流媒体、RK 透传）**、**RK 嵌入式（GPIO 等）**、**Qt**。  
> 语法细节不必按教材拆太碎——合并速查即可；精力投向 **OO + STL/算法 + OS/网络/组成 + 构建与系统编程 + 三条业务线专项**。  
> 下文自「一」起为 **知识文档架构**（合并大章，可自行填充笔记）。

---

## 学习路线

**层级 A：C++ 语法一次性通关（低优先级维护）**

- **目标**：把「和 C 重叠的部分」快速掠过；把「C++ 特有」列成清单：`namespace`、引用、重载、`new/delete`、类、模板、异常、`constexpr`、lambda、智能指针等。
- **产出**：一份个人 **语法速查表**（一页纸 mindmap 亦可）；遇到问题查 [cppreference](https://en.cppreference.com/) 能定位到条目。

**层级 B：面向对象 + 资源语义（JNI/Qt 写类时常用）**

- **目标**：构造/析构、RAII、拷贝与移动、继承与多态、运算符重载；**JNI 里 C++ 封装 Native 资源**、Qt 里对象树与信号槽之前的基础。
- **产出**：能写「资源随生命周期释放」的类；理解值 / 引用 / 指针在接口设计上的选择。

**层级 C：STL 深度绑定「数据结构 + 算法分析」**

- **目标**：每个经典结构对应 STL 容器/适配器；每个算法类别对应 `<algorithm>` / 容器成员算法；复杂度与迭代器类别挂钩。
- **产出**：刷题或工程选型时能说出「为何用 `deque` / `priority_queue` / `unordered_map`」以及均摊 / 最坏情形。

**层级 D：408 三门（操作系统、计组、计算机网络）**

- **目标**：建立统考教材级框架，再在你关心的点上加深：**线程/同步/文件/调度**（对接 Linux 与 NDK）、**协议栈与实时流媒体相关协议**（对接 JNI 业务）、**存储器层次与 ARM/RK 常识**（对接嵌入式）。
- **产出**：能用自己的话画进程状态图、三级调度、虚拟内存概念图；能简述 TCP/UDP 差异及适用场景。

**层级 E：编译 / 链接 / 构建（工程刚需）**

- **目标**：理解预处理、编译、汇编、链接；符号与 ABI；**Makefile** 或 **CMake** 至少熟练一种；Android **NDK** / **交叉编译** 有概念。
- **产出**：能从零拉起一个小型多文件 C++ 工程；JNI `.so` 构建流程能跟着文档走通。

**层级 F：Linux 系统编程（与 C++ 并用）**

- **目标**：文件 IO、`mmap`、进程/线程、管道、Socket、信号、`poll`/`epoll` 概念；与 `std::thread` / `std::filesystem` 对照。
- **产出**：能阅读典型的 POSIX + C++ 混合代码（流媒体路径上很常见）。

**层级 G：三条业务线专项（按需并行）**

| 方向 | 核心抓手 |
|------|----------|
| **Android JNI** | `JNI_OnLoad`、线程绑定、`jbyteArray`/Direct Buffer、异常检查、与 Java/Kotlin 边界；流媒体：**编码帧、时间戳、环形缓冲、背压**；RK：**硬件编解码/透传接口按厂商文档**。 |
| **RK 嵌入式** | 设备树 / sysfs / `gpiod` 或厂商 SDK、权限与 SELinux 常识、**最小化 C++ 子集 + C 驱动接口**。 |
| **Qt** | 事件循环、对象树、`moc`、信号槽、与 C++11+ 特性配合；跨平台与嵌入式 Qt 取舍。 |

**层级 H：持续补充**

- 调试（gdb、lldb、AddressSanitizer）、性能（perf、simple profiler）、代码规范（clang-tidy）、版本管理。

---

## 一、C++ 语法总览（合并章：几乎不必单独啃书）

> 把「全书语法」收成一块：需要时查阅，日常以写代码 + 速查为主。

- **与 C 的差异清单**：编译单元、链接、`extern "C"`、名字修饰、类型检查、`bool`、引用、默认参数、`namespace`
- **类型与运算**：基本类型、`const`/`constexpr`、类型推导（`auto`、`decltype`）、转换、`enum class`
- **语句与表达式**：范围 for、异常语法、`noexcept`（概述）
- **函数**：重载、内联、`constexpr`、lambda（捕获、泛型 lambda）
- **内存**：栈/堆、`new`/`delete`、对齐（简要）
- **模板入门**：函数模板、类模板、概念 constraints（C++20，可选）
- **其他语法**：友元、`static` 成员、嵌套类型、`using` 声明与别名

---

## 二、面向对象与类设计（C++ 特有的抽象层）

- **类与封装**：访问控制、`this`、构造 / 析构、初始化列表、委托构造
- **运算符重载**：成员与非成员、对称性、`<<`/`>>` 与 IO
- **继承与多态**：虚函数、纯虚与接口、`override`/`final`、多重继承（慎用场景）
- **拷贝控制**：三五法则、移动语义、`= default` / `= delete`
- **RAII 与异常安全**：资源即对象；JNI/Qt 里封装句柄的典型写法

---

## 三、数据结构与算法分析 · 深度结合 STL

> **用法**：复习 DS&A 时 **强制对应** STL；分析复杂度时用容器/算法的迭代器与操作代价。

**线性结构 ↔ STL**

- 数组 / 动态数组 ↔ `array`、`vector`（扩容均摊）、`deque`
- 链表 ↔ `list`、`forward_list`
- 栈 / 队列 ↔ `stack`、`queue`、`priority_queue`（底层容器）
- 字符串 ↔ `string`、`string_view`

**树与堆 ↔ STL**

- BST / 自平衡树思路 ↔ `map`、`multimap`、`set`、`multiset`
- 堆 ↔ `priority_queue`；自定义比较器与稳定性直觉

**哈希 ↔ STL**

- 哈希表 ↔ `unordered_map`、`unordered_set`；负载因子与 rehash；自定义哈希与相等

**图论与其他**

- 邻接表 / 矩阵的存储选型；与容器组合实现（非 STL 单一容器）
- 并查集、Trie 等常用「手写」结构与何时不必造轮子

**算法 ↔ `<algorithm>` 与复杂度**

- 排序 / 二分 / 堆操作：`sort`、`nth_element`、`lower_bound`…
- 遍历与变换：`for_each`、`transform`、`accumulate`…
- 区间与迭代器：迭代器类别（输入、随机访问…）与算法前提条件

**实务**

- 时间与空间复杂度笔记；摊还分析（与 `vector::push_back` 等对照）
- 竞赛/面试题型与 STL 选型速查表（自建）

---

## 四、操作系统（参考 408：王道 / 汤小丹等教材脉络）

> 与 **NDK 线程**、**RK/Linux 运行时**、流媒体缓冲策略强相关。

- **操作系统概述**：功能、特征、接口
- **进程与线程**：状态与转换、PCB、线程模型；与 `pthread` / `std::thread` 对照
- **调度**：调度算法、上下文切换、优先级（概念）
- **同步与互斥**：信号量、 mutex、条件变量、读写锁、经典问题；与 C++ 并发库对照
- **死锁**：条件与避免、检测
- **内存管理**：分页、分段、虚拟内存、页面置换；与 mmap、缓存友好编程的联系
- **文件系统**：目录结构、索引、缓存；与 Linux VFS、`open/read/write` 的联系
- **设备管理 / IO**：缓冲、Spooling（概念）；块设备与字符设备常识
- **虚拟化与云（408 扩展）**：按需

---

## 五、计算机网络（408 脉络 + 流媒体相关）

- **体系结构与参考模型**：OSI、TCP/IP
- **物理层 / 链路层**：帧、差错控制（概念）；以太网、WLAN 常识
- **网络层**：IPv4、子网划分、路由基础；ICMP 常识
- **传输层**：TCP / UDP、端口、可靠传输与流量控制；**流媒体场景下的选型**
- **应用层**：DNS、HTTP；**RTMP / RTSP / RTP（按需深入）**
- **网络安全基础**：TLS 常识（JNI 拉流有时涉及 HTTPS）

---

## 六、计算机组成原理（408 脉络 + ARM/RK 直觉）

- **数制与编码**：定点、浮点（概念）
- **CPU**：指令格式、寻址、CISC/RISC；ARM 与 x86 差异常识
- **存储器层次**：Cache、主存、局部性；与性能优化
- **总线与 IO**：中断、DMA（概念）
- **指令流水线**：冒险（概念）

---

## 七、编译原理与构建（与 C++ 强绑定）

**编译原理（实用向）**

- **编译管线**：预处理 → 编译 → 汇编 → 链接
- **前端相关**：词法/语法/语义（概念层）；与错误信息阅读
- **优化与内联**：`-O2` 常识；与 `inline`、`constexpr` 的关系（不混淆）
- **链接**：符号解析、重定位、静态库 / 动态库；**ABI**、名称修饰；JNI `.so` 加载路径

**Makefile**

- **规则**、变量、模式规则、伪目标
- **多目录**、依赖生成（可选 `gcc -MM`）
- **与 NDK**：`ndk-build` / 手写 Makefile 交叉编译变量（`CC`、`CFLAGS`、`LDFLAGS`）

**CMake（推荐主修）**

- **基本**：`project`、`add_executable`、`add_library`、`target_*`
- **跨平台**：生成器、工具链文件（**交叉编译 ARM/RK**）
- **与 Android**：`ExternalNativeBuild`、传入 ABI、`find_library` log

---

## 八、Linux 系统编程（与 C/C++ 混合代码）

- **文件与 IO**：`open`、`read`、`write`、`lseek`、`mmap`、`fcntl`
- **进程**：`fork`、`exec`、`wait`、进程组
- **线程**：`pthread`；与 `std::thread` 映射
- **同步**：`pthread_mutex`、`pthread_cond`、屏障（概念）
- **进程间通信**：管道、FIFO、`shm`、消息队列、Socket（概要）
- **信号**：常用信号与异步信号安全
- **网络编程**：Socket 基础（与流媒体上层协议衔接）
- **epoll / 事件驱动**（服务端或高性能路径）：概念级

---

## 九、专项：Android JNI 与 Native（流媒体 / RK 透传）

- **JNI 基础**：类型映射、`RegisterNatives`、局部引用 / 全局引用、`DeleteLocalRef`
- **线程**：`AttachCurrentThread` / `DetachCurrentThread`；与音频线程、解码线程
- **缓冲区**：Direct `ByteBuffer`、`GetPrimitiveArrayCritical` 取舍与性能
- **异常**：`ExceptionCheck` / `ExceptionDescribe`；不在 Native 乱抛未处理异常
- **流媒体**：时间戳、环形队列、线程模型（采集 / 编码 / 发送）；丢帧与背压
- **RK 相关**：按 **Rockchip** 文档对接 MPP、GPU/VPU、透传路径（笔记占位：**随 SDK 更新**）

---

## 十、专项：RK 嵌入式开发（GPIO 等）

- **内核与用户态**：设备节点、`ioctl`（概念）
- **GPIO**：sysfs 路径、`libgpiod`、旧 API 对照；权限与 udev（常识）
- **设备树**（概念）：硬件描述与驱动匹配
- **C++ 使用策略**：HAL 层封装、尽量少异常；与厂商 BSP 目录结构

---

## 十一、专项：Qt 开发

- **核心**：元对象系统、`moc`、信号与槽、事件循环
- **模块**：Widgets / Quick（按方向）；资源文件 `.qrc`
- **与 C++**：智能指针与 QObject 所有权；线程与 `moveToThread`
- **构建**：CMake + Qt、交叉编译到 ARM（RK 屏端等场景）

---

## 十二、C++ 其他知识与扩展索引

- **并发**：`std::thread`、`mutex`、`condition_variable`、原子操作与内存序（进阶）
- **标准库杂项**：`chrono`、`random`、`regex`、`filesystem`（C++17）
- **惯用法**：《Effective C++》《Effective Modern C++》主题索引（自建 checklist）
- **互操作**：与 C、Python、其他 DLL/so；**JNI** 已单列
- **调试与 Sanitizer**：ASan、TSan、UBSan；Android `ndk-stack`
- **版本**：选定项目用的 **C++ 标准**（NDK / RK 工具链支持表）

---

## 附录（笔记占位）

- 个人语法一页纸 / 速查链接
- STL ↔ 数据结构对照表（完成度自检）
- 408 三门思维导图路径
- NDK / RK SDK / Qt 版本与工具链记录（避免环境漂移）
