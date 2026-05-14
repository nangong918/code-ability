# Android

> **定位**：多年 Android 应用与系统侧经验；本文是 **总览与索引**，细节分散在专题文档。  
> **范围约定**：**Flutter 归入 Android 知识树**；**嵌入式 / 定制 Android（AOSP、Framework、BSP）** 与常规 App 分流——深度见 [AndroidFramework.md](../AndroidFramework/AndroidFramework.md)。  
> **UI 分流**：**XML / View / Fragment 传统体系** 在本总览中提纲；**Jetpack Compose** 见 [JetpackCompose.md](Kotlin/JetpackCompose/JetpackCompose.md)；**Flutter** 见 [Flutter.md](Flutter/Flutter.md)，Dart 语言见 [Dart.md](../Language/Dart.md)。

---

## 导读：知识地图

| 主题 | 详述文档 |
|------|----------|
| 进程间通信（Binder、AIDL、Provider 等） | [IPC.md](IPC.md)（全量） |
| Native / JNI / NDK | [JNI.md](JNI.md)（全量）；C++ 侧配合 [C++.md](../Language/C++.md) |
| Kotlin 语言与协程 | [Kotlin.md](../Language/Kotlin.md)（重点全量） |
| Dart（Flutter） | [Dart.md](../Language/Dart.md) |
| Jetpack Compose | [JetpackCompose.md](Kotlin/JetpackCompose/JetpackCompose.md)（提纲） |
| Kotlin Multiplatform | [KMP.md](Kotlin/KMP/KMP.md)（提纲） |
| Flutter 框架 | [Flutter.md](Flutter/Flutter.md)（提纲） |
| Framework / 嵌入式 / AOSP 定制 | [AndroidFramework.md](../AndroidFramework/AndroidFramework.md)（全量梳理） |

---

## 平台分层（心智模型）

**应用层**：系统应用、三方 APK、WebView、Flutter Engine 宿主等。

**Android Framework（Java/Kotlin API）**：`Activity`、`Service`、Window、Package、Resource 等对外契约；内部由 **system_server** 中的系统服务实现（见 Framework 专文）。

**Native**：`SurfaceFlinger`、`mediaserver` / `audio` / `camera` 相关进程、`libbinder`、`hwbinder`、JNI 桥。

**ART**：DEX/字节码、JIT/AOT、GC（应用进程内）。

**HAL**：硬件抽象接口（Camera、Audio、Graphics…），对接内核与驱动；Project Treble 后 **vendor** 分区与 **System** 边界清晰。

**Linux Kernel**：驱动、内存、调度、Binder 驱动等。

---

## 语言与运行时

- **Kotlin**：首选；协程、DSL、空安全——详见 [Kotlin.md](../Language/Kotlin.md)。
- **Java**：互操作、遗留模块、部分 SDK 示例仍为 Java。
- **C/C++**：JNI/NDK、Rust（少量场景）——JNI 见 [JNI.md](JNI.md)。

---

## 四大组件（提纲）

**Activity**：界面容器、任务栈、`launchMode`、生命周期、`onSaveInstanceState`、进程被杀恢复。

**Service**：前台 Service（通知与后台限制）、`bindService`、`IntentService`/WorkManager 替代关系。

**BroadcastReceiver**：显式/隐式、`ordered`、`sticky`（已弱化）、LocalBroadcastManager（废弃）→ 应用内用 LiveData/Flow 等。

**ContentProvider**：跨进程数据共享、URI、`CRUD`、与 Room 作为对外暴露层的组合。

细节与 IPC 交叉：**跨进程**必读本仓库 [IPC.md](IPC.md)。

---

## UI：传统 XML / View 体系（写在本总览）

**布局**：`ConstraintLayout`、`LinearLayout`、`RelativeLayout`、`FrameLayout`、`RecyclerView` + `ListAdapter`/`DiffUtil`。

**自定义绘制**：`View`/`ViewGroup` 测量布局绘制、`Canvas`、硬件加速、`SurfaceView`/`TextureView`（视频、相机预览）。

**动画**：`PropertyAnimation`、`Transition`、`MotionLayout`。

**主题与资源**：`themes`、`styles`、限定符、`VectorDrawable`、夜间模式。

**无障碍与国际化**：`contentDescription`、RTL、`AutoSize Text`。

Jetpack 辅助：`Fragment`、`Navigation`、`ViewBinding`/`DataBinding`、`Lifecycle`、`ViewModel`、`Paging`、`Room`（数据层常在 MVVM/MVI 中与 UI 解耦）。

---

## Jetpack Compose

声明式 UI、重组、`State`、副作用、`Navigation Compose` 等——提纲见 [JetpackCompose.md](Kotlin/JetpackCompose/JetpackCompose.md)。

---

## Flutter（归入 Android 大类）

引擎、Widget、`Platform Channel`、混合栈、与 Android 嵌入——提纲见 [Flutter.md](Flutter/Flutter.md)；语言层见 [Dart.md](../Language/Dart.md)。

---

## 架构模式（表现层）

**MVC**：Activity 过胖；实践中少用纯 MVC。

**MVP**：`View` 接口 + `Presenter`；接口成本高，新项目偏少。

**MVVM**：`ViewModel` + `LiveData`/`Flow` + DataBinding/ViewBinding；与 Jetpack 一致性好。

**MVI**：单向数据流、`State` 归约、副作用隔离；Kotlin `Flow` 表达力强。

**Clean Architecture / 分层**：Domain—Data—UI，与具体模式可组合。

笔记可按项目沉淀「包结构 + 事件流」一页纸。

---

## IPC（进程间通信）

Binder、AIDL、`Messenger`、`ContentProvider`、Socket、`SharedMemory`、Binder 限制与调试——**全量**见 [IPC.md](IPC.md)。

---

## JNI 与 Native

ABI、`CMake`、`ndk-build`、`JNI_OnLoad`、线程、数组与 Direct Buffer——**全量**见 [JNI.md](JNI.md)。

---

## 跨平台（Android 视角）

| 技术 | 文档 |
|------|------|
| Flutter | [Flutter.md](Flutter/Flutter.md)、[Dart.md](../Language/Dart.md) |
| Compose Multiplatform（UI 共享） | [JetpackCompose.md](Kotlin/JetpackCompose/JetpackCompose.md) 中跨平台索引 |
| Kotlin Multiplatform（逻辑共享） | [KMP.md](Kotlin/KMP/KMP.md) |

---

## 系统特性与安全（提纲）

- **权限**：运行时权限、特殊权限、安装未知来源、`MANAGE_EXTERNAL_STORAGE` 等演变。
- **后台限制**：Doze、待机桶、前台服务类型（媒体、位置等）。
- **存储**：分区存储（Scoped Storage）、MediaStore、SAF。
- **多进程**：自定义进程名、`Multidex`、应用属性。
- **网络安全**：默认禁止明文流量、`Network Security Config`。
- **BIOMETRIC / Keystore**：密钥与硬件绑定（提纲）。

---

## 嵌入式 Android 与 Framework 定制

启动流程修改、系统应用、HAL/Framework 交界、SELinux、分区——**系统侧全量**见 [AndroidFramework.md](../AndroidFramework/AndroidFramework.md)。

---

## 附录：文档维护约定

- 应用开发细节优先记在业务仓库；此处保持 **导航 + 术语一致**。
- 路径均为相对于 `Knowledge` 仓库内 Markdown 的相对链接，复制时注意根目录。
