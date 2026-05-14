# Dart

> Flutter 主力语言；本文服务 **Flutter 文档树**（见 [Flutter.md](../Android/Flutter/Flutter.md)）。语法细节以 [Dart 官方文档](https://dart.dev/language) 为准，此处为结构化索引。

---

## 语言概览

- **类型**： sound null safety；`?`、`!`、`late`；类型推断。
- **变量**：`var`、`final`、`const`；编译时常量与运行时常量区分。
- **函数**：命名参数、可选位置参数、默认实参、**一等函数**。
- **面向对象**：mixin、`extension`、工厂构造、`abstract`/`implements`。

---

## 异步模型

- **`Future`** / **`async`/`await`**：事件循环上执行；**勿在 compute 密集任务阻塞 isolate**。
- **Isolate**：无共享内存并发；**消息传递**；`Isolate.spawn`、`ReceivePort`。
- **`compute()`**：后台 isolate 跑纯函数（解码 JSON、大图处理等）。

---

## 集合与泛型

- **`List`/`Map`/`Set`** 字面量；**展开** `...`、`...?`。
- **泛型**：协变/逆变规则与 Java/Swift 对照笔记（按需）。

---

## Flutter 侧常用

- **`Widget` 与不可变**：配置对象；状态外提。
- **`BuildContext`**：层级敏感；**InheritedWidget** 查找。
- **渲染**：布局约束向下、尺寸向上（与 Compose 心智对照可在笔记里一行对照）。

---

## 工具链

- **`dart analyze` / `dart format`**、`pub` 依赖解析。
- **FFI**：`dart:ffi` 对接 Native（高级）；多数场景优先 **Platform Channel**。

---

## 延伸阅读

- [Flutter.md](../Android/Flutter/Flutter.md)
- [Android.md](../Android/Android.md)
