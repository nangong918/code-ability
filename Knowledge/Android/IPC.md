# IPC

> Android 进程隔离：多数组件跑在独立进程，跨进程依赖 **Binder** 机制及建立在之上的 API（AIDL、`Messenger`、`ContentProvider` 等）。本文面向「全栈梳理」，调试时可配合 `dumpsys`、`binder` 统计。

---

## Binder 是什么

- **动机**：Linux 传统 IPC（管道、Socket、共享内存）在 **安全校验、拷贝次数、接口描述** 上对移动端不友好；Binder 在内核提供驱动（`/dev/binder`，binderfs），框架提供 **抽象层**：BBinder/BpBinder、`Parcel`、`IBinder.transact()`。
- **特点**：基于 **能力 / 令牌** 的引用（不是裸指针）、支持同步 RPC 与 **one-way**、与 Java 层 `Binder` 类衔接。
- **架构**：Client（Bp）— Binder 驱动 — Server（Bn）；`servicemanager` / `hwservicemanager` / `vndservicemanager`（按 Treble 世代）负责 **按名字注册与查找** 服务。

---

## Binder 事务与限制

- **Parcel**：序列化容器；跨进程传递 **基本类型、Parcelable、IBinder、文件描述符（fd）** 等。
- **Binder 缓冲区**：单次事务数据量有上限（历史上约 **1MB** 量级，具体随版本与共享缓冲区策略变化）；大图、长数组应 **分块**、用 **ASHMEM / SharedMemory**、或 **管道式流式**。
- **同步调用栈**：主线程执行 Binder **同步** 可能导致 ANR；避免 UI 线程阻塞。
- **死亡通知**：`IBinder.linkToDeath()`，服务进程崩溃时客户端可感知。

---

## AIDL（Android Interface Definition Language）

**用途**：定义接口并生成 Java/C++/Rust stub，完成跨进程方法调用。

**文件**：`*.aidl`；支持 **定向标记**：`in`、`out`、`inout`（对象需实现 `Parcelable`）。

**生成物**：Java 侧代理与 Stub；编译期生成。

**线程模型**：Stub 默认回调可能在 **Binder 线程池**（非主线程）；若更新 UI 需切回主线程。

**Listenable future / 单向**：`oneway` 接口方法：异步 fire-and-forget，**无返回值**，顺序弱保证。

**版本演进**：稳定接口可考虑 **AIDL + 版本字段**、或 Android 各版本中对 **Parcel 写入顺序** 保持兼容。

---

## Messenger + Handler

**场景**：跨进程 **消息队列** 式通信，比手写 AIDL 轻。

**原理**：底层仍是 Binder；`Messenger` 持有指向服务端 `Handler` 的 `IBinder`。

**限制**：多为 **单向 / 请求-响应** 模式；大数据同样受 Parcel 限制。

---

## ContentProvider 作为 IPC

- **跨进程 CRUD**：通过 `ContentResolver` + URI；底层走 Binder。
- **批量**：`bulkInsert`、`applyBatch` 减少往返。
- **权限**：`android:readPermission` / `writePermission`、URI 权限（临时授权）、`grantUriPermission`。
- **与 Room**：Room 通常同进程；对外暴露可写 Provider 包装。

---

## Broadcast 的进程边界

- **跨进程广播**：隐式广播在 Android 8+ **后台限制** 大量收紧；系统广播仍按文档可用。
- **有序广播**：可被中止（`abortBroadcast`），仍有同步栈含义。
- **本地替代**：应用内用 **SharedFlow / RxBus（慎用）/ 直接回调**，避免滥用全局广播。

---

## Socket 与 LocalSocket

- **TCP/UDP**：与其他设备或本机其他进程；注意权限与 SELinux。
- **LocalSocket（UNIX domain）**：本机进程；仍需权限匹配。
- **适用**：流媒体、大量字节流、已与 Binder 不适合的大数据通路（仍要注意 selinux policy）。

---

## SharedMemory / Ashmem

- **用途**：大块缓冲区 **零拷贝或低开销共享**（音视频、图像）；配合 **同步原语**（`ParcelFileDescriptor`、信号量或 Atomic 等）。
- **注意**：生命周期与 **fd 传递**、权限、同步错误易导致 **数据撕裂**。

---

## Parcelable 与 Serializable

- **Parcelable**：Android 推荐；显式 `writeToParcel` / `CREATOR`。
- **Serializable**：反射重，跨进程可用但不优先。
- **版本**：字段增删要有 **兼容策略**（默认忽略未知字段等）。

---

## HIDL / AIDL HAL（了解与 IPC 关系）

- **Framework ↔ Vendor**：硬件相关通过 HAL；旧 HIDL、新 **AIDL HAL**（Treble 演进）。
- **与 App 开发**：日常少直接写；系统工程师调 HAL；App 通过 Java API 间接使用。

---

## 调试与诊断

- **`dumpsys activity|package|media`**：服务状态。
- **`binder` 事务失败**：`TransactionTooLargeException`、`-32` 等错误码。
- **systrace / Perfetto**：Binder 阻塞片段。
- **日志**：`tags` 含 `ActivityManager`、`PackageManager` 等。

---

## SELinux 与 IPC

- **域（domain）**：进程标签；跨域访问需 **allow 规则**。
- **定制系统**：改 IPC 路径可能要在 `sepolicy` 中放行 **unix_socket**、**binder**、**fd** 等。

---

## 选型速查

| 需求 | 常见方案 |
|------|----------|
| 频繁 RPC、强类型 | AIDL |
| 简单消息、不需双向复杂接口 | Messenger |
| 共享数据库式数据 | ContentProvider |
| 大块数据 / 流 | Socket、管道、SharedMemory + 同步 |
| 仅应用内模块解耦 | 不用 Binder；接口 + DI |

---

## 延伸阅读（仓库内）

- Framework 层 Binder 注册与 **system_server**：见 [AndroidFramework.md](../AndroidFramework/AndroidFramework.md)。
- JNI 与 Native 回调跨线程：见 [JNI.md](JNI.md)。
