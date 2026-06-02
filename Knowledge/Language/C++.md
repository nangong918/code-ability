# C++

---

## 学习路线

**层级 A：C++ 语法一次性通关**

- **目标**：把「和 C 重叠的部分」快速掠过；把「C++ 特有」列成清单：`namespace`、引用、重载、`new/delete`、类、模板、异常、`constexpr`、lambda、智能指针等。
- **标准**：一份个人 **语法速查表**（一页纸 mindmap 亦可）；遇到问题查 [cppreference](https://en.cppreference.com/) 能定位到条目。

**层级 B：面向对象 + 资源语义**

- **目标**：构造/析构、RAII、拷贝与移动、继承与多态、运算符重载；
- **标准**：能写「资源随生命周期释放」的类；理解值 / 引用 / 指针在接口设计上的选择。

**层级 C：STL 深度绑定「数据结构 + 算法分析」**

- **目标**：每个经典结构对应 STL 容器/适配器；每个算法类别对应 `<algorithm>` / 容器成员算法；复杂度与迭代器类别挂钩。
- **标准**：刷题或工程选型时能说出「为何用 `deque` / `priority_queue` / `unordered_map`」以及均摊 / 最坏情形。

**层级 D：408 三门（操作系统、计组、计算机网络）**

- **目标**：建立统考教材级框架，再在你关心的点上加深：**线程/同步/文件/调度**（对接 Linux 与 NDK）、**协议栈与实时流媒体相关协议**（对接 JNI 业务）、**存储器层次与 ARM/RK 常识**（对接嵌入式）。
- **标准**：能用自己的话画进程状态图、三级调度、虚拟内存概念图；能简述 TCP/UDP 差异及适用场景。

**层级 E：编译 / 链接 / 构建**

- **目标**：理解预处理、编译、汇编、链接；符号与 ABI；**Makefile** 或 **CMake** 至少熟练一种；Android **NDK** / **交叉编译** 有概念。
- **标准**：能从零拉起一个小型多文件 C++ 工程；JNI `.so` 构建流程能跟着文档走通。

**层级 F：Linux 系统编程（与 C++ 并用）**

- **目标**：文件 IO、`mmap`、进程/线程、管道、Socket、信号、`poll`/`epoll` 概念；与 `std::thread` / `std::filesystem` 对照。
- **标准**：能阅读典型的 POSIX + C++ 混合代码（流媒体路径上很常见）。

**层级 G：三条业务线专项（按需并行）**

| 方向 | 核心抓手 |
|------|----------|
| **Android JNI** | `JNI_OnLoad`、线程绑定、`jbyteArray`/Direct Buffer、异常检查、与 Java/Kotlin 边界；流媒体：**编码帧、时间戳、环形缓冲、背压**；RK：**硬件编解码/透传接口按厂商文档**。 |
| **RK 嵌入式** | 设备树 / sysfs / `gpiod` 或厂商 SDK、权限与 SELinux 常识、**最小化 C++ 子集 + C 驱动接口**。 |
| **Qt** | 事件循环、对象树、`moc`、信号槽、与 C++11+ 特性配合；跨平台与嵌入式 Qt 取舍。 |


## 一、C++ 语法总览

- **与 C 的差异清单**：编译单元、链接、`extern "C"`、名字修饰、类型检查、`bool`、引用、默认参数、`namespace`
- **类型与运算**：基本类型、`const`/`constexpr`、类型推导（`auto`、`decltype`）、转换、`enum class`
- **语句与表达式**：范围 for、异常语法、`noexcept`（概述）
- **函数**：重载、内联、`constexpr`、lambda（捕获、泛型 lambda）
- **内存**：栈/堆、`new`/`delete`、对齐（简要）
- **模板入门**：函数模板、类模板、概念 constraints（C++20，可选）
- **其他语法**：友元、`static` 成员、嵌套类型、`using` 声明与别名

### C++ 关键字

#### 一、基本数据类型

**关键字**：bool, char, wchar_t, short, int, long, float, double, void

**含义用途**：C++基础原生数据类型，用于定义变量的数据存储格式。
包含布尔类型、单字节字符、宽字符、短整型、普通整型、长整型、单精度浮点型、双精度浮点型，void 代表无类型，
常用于无返回值函数或空指针定义。

**实例代码**

```cpp
bool flag = true;
char ch = 'A';
char* str = "hello"; 
wchar_t wch = L'中';
short s_num = 10;
int num = 100;
long l_num = 10000;
float f_pi = 3.14f;
double d_pi = 3.1415926;
void func() {}
```

#### 二、类型修饰符

**关键字**：signed, unsigned, const, volatile, mutable

**含义用途**：用于修饰变量、类成员的属性，限定数据的读写特性、符号特性与内存特性。
signed/unsigned 区分数值正负；
const 定义只读常量；
volatile 告知编译器变量值可能被外部修改，禁止优化；
mutable 允许类内常量成员在 const 成员函数中修改。

**实例代码**

```cpp
const int MAX_SIZE = 100;
unsigned int u_val = 255;
signed int s_val = -50;
// 嵌入式开发中，会遇到编译器将int优化为常量，此时就要用volatile标注为可变量
volatile int status = 0;

class Test {
public:
    // 常量对象可修改
    mutable int cnt;
};
```

#### 三、标准类型转换

**关键字**：static_cast, dynamic_cast, const_cast, reinterpret_cast

**含义用途**：C++专属四种安全、规范的类型转换方式，替代C语言强制转换。
static_cast 实现常规静态类型转换；
dynamic_cast 用于多态父子类安全转换；
const_cast 专门移除/添加常量属性；
reinterpret_cast 实现内存二进制重解释转换。

**实例代码**

```cpp
// 静态转换
int a = 10;
double b = static_cast<double>(a);

// 常量转换
const int c_num = 20;
int* change_num = const_cast<int*>(&c_num);

// 重解释转换
int* ptr = reinterpret_cast<int*>(0x1000);
```

#### 四、存储与链接属性

**关键字**：auto, register, static, extern, explicit, export

**含义用途**：控制变量、函数的存储位置、生命周期、作用域及跨文件链接特性。
auto 自动推导变量类型；
register 建议变量存储在寄存器；
static 修饰静态变量/函数，延长生命周期、限定文件作用域；
extern 声明外部全局变量/函数；
explicit 禁止构造函数隐式类型转换；
export 用于导出模板跨文件使用。

**实例代码**

```cpp
auto val = 3.14; // 自动推导为double
register int cnt = 0; // 寄存器变量
static int static_val = 1; // 静态局部变量
extern int g_val; // 声明外部全局变量

class Demo {
public:
    explicit Demo(int a) {} // 禁止隐式转换
};
```

#### 五、流程控制语句

**关键字**：if, else, for, while, do, switch, case, default, break, continue, goto

**含义用途**：控制程序代码执行逻辑与流程，实现条件判断、循环遍历、分支选择、流程跳转。是程序逻辑编写的核心关键字，支撑所有业务逻辑流程。

**实例代码**

```cpp
// 条件判断
bool flag = true;
if (flag) {} else {}

// 循环语句
for (int i = 0; i < 5; i++) {}
int j = 0;
while (j < 3) j++;
do {} while (false);

// 分支语句
int num = 2;
switch (num) {
    case 1: break;
    case 2: break;
    default: break;
}
```

#### 六、异常处理机制

**关键字**：try, catch, throw

**含义用途**：C++标准异常处理体系，用于捕获、抛出、处理程序运行时异常，避免程序直接崩溃，提升代码健壮性。try 包裹监控代码，throw 主动抛出异常，catch 匹配并处理对应异常。

**实例代码**

```cpp
try {
    int a = 10, b = 0;
    if (b == 0) throw "除数不能为0";
}
catch (const char* err) {
    // 捕获并处理异常
}

```


#### 七、面向对象核心

**关键字**：class, struct, union, enum, public, protected, private, friend, virtual

**含义用途**：支撑C++面向对象编程特性，用于定义类、结构体、枚举、共用体；
控制类成员访问权限；
定义友元打破访问限制；
定义虚函数实现多态特性。

**实例代码**

```cpp
enum Color { RED, GREEN, BLUE };
struct Student { int id; };
union Data { int a; char b; };

class Base {
private:
    int private_val; // 私有成员
protected:
    int pro_val; // 保护成员
public:
    virtual void func() {} // 虚函数
    friend class Test; // 友元类
};
```


#### 八、函数工具关键字

**关键字**：return, inline, operator, sizeof, this

**含义用途**：辅助函数定义、调用与运算重载。
return 用于函数返回值；
inline 定义内联函数减少调用开销；
operator 实现运算符重载；
sizeof 计算数据类型/变量占用内存字节数；
this 指向当前类对象的指针。

**实例代码**

```cpp
inline int add(int a, int b) {
    return a + b;
}

class Calc {
public:
    int operator+(const Calc& other) { return 1; }
    void show() {
        cout << sizeof(int) << endl;
        this->add(1,2);
    }
};
```


#### 九、命名空间与泛型模板

**关键字**：namespace, template, typename, using

**含义用途**：解决命名冲突、支撑泛型编程。
namespace 定义命名空间隔离代码；
using 引入命名空间或类型别名；
template 定义泛型模板；
typename 标识模板中的类型参数。

**实例代码**

```cpp
namespace MyCode {
    int num = 100;
}
using namespace MyCode;

template<typename T>
T getMax(T a, T b) {
    return a > b ? a : b;
}
```

#### 十、类型别名与类型查询

**关键字**：typedef, typeid

**含义用途**：typedef 用于为已有数据类型自定义别名，简化代码书写；
typeid 用于运行时获取变量、对象的具体类型信息，常用于类型判断与调试。

**实例代码**

```cpp
typedef int Integer;
Integer a = 10;

// 获取类型名称
cout << typeid(a).name() << endl;
```

#### 十一、动态内存管理

**关键字**：new, delete

**含义用途**：用于堆区动态内存的申请与释放。new 手动在堆内存分配空间并初始化；
delete 释放 new 申请的堆内存，避免内存泄漏。

**实例代码**

```cpp
// 单个内存申请释放
int* p = new int(5);
delete p;

// 数组内存申请释放
int* arr = new int[10];
delete[] arr;

```

#### 十二、底层汇编支持

**关键字**：asm

**含义用途**：允许在C++代码中直接嵌入汇编语言指令，用于底层硬件操作、性能优化、系统级开发场景，普通业务开发极少使用。

**实例代码**

```cpp
// 嵌入空汇编指令
asm("nop");



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
