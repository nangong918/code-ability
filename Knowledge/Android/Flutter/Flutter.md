# Flutter

> **定位**：归入 Android 知识树；宿主可为 Android（本文默认）、iOS、Desktop。细节语言层见 [Dart.md](../../Language/Dart.md)；总导航见 [Android.md](../Android.md)。

---

## 引擎与架构（提纲）

**Flutter Engine**：Skia / Impeller（渲染后端依版本）、Dart VM、平台通道；与宿主 OS 通过 **embedder** 衔接。

**Framework（Dart）**：`Widgets`、`Rendering`、`Painting`、`Semantics` 分层；**一切皆 Widget** 的组合模型。

**Build 模式**：`debug` / `profile` / `release`；性能分析用 profile。

---

## UI：写在 Flutter 文档（本页）

- **Widget 分类**：`StatelessWidget`、`StatefulWidget`；`InheritedWidget` / `InheritedModel`；`RenderObjectWidget` 深入。
- **Element / RenderObject**：三棵树心智模型（调试布局必读）。
- **布局**：`Flex`、`Stack`、`ListView`/`Sliver`、约束传递。
- **主题**：`Theme`、`Material`/`Cupertino`、暗黑模式。

XML / Jetpack Compose 对照见 [Android.md](../Android.md) 导读表。

---

## 状态管理（索引）

按项目选型：`setState`、**Provider**、**Riverpod**、**Bloc**、**GetX**（争议）、`ValueNotifier` 等。

笔记建议：每种模式一页「数据流 + 测试策略」。

---

## 与 Android 宿主

- **Platform Channel**：`MethodChannel`、`EventChannel`、`BasicMessageChannel`；二进制协议与线程约定。
- **混合栈**：Flutter 内嵌 Native `Fragment` / Activity，路由与返回栈协同。
- **后台限制**：仍遵守 Android 前台服务与省电策略；Flutter 侧 isolate 不替代系统策略。

---

## 构建与依赖

- **`pubspec.yaml`**：依赖、资源、字体。
- **Gradle 集成**：Flutter module / AAR；ABI、Proguard、签名与常规 Android 一致。

---

## 延伸阅读

- [Dart.md](../../Language/Dart.md)
- [JNI.md](../JNI.md)（Platform Channel 底层或纹理集成 Native 时）
