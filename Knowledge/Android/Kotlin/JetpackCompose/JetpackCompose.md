# Jetpack Compose

> **声明式 UI** 写在本文；传统 XML/View 体系写在 [Android.md](../../Android.md)。Kotlin 语言与协程见 [Kotlin.md](../../../Language/Kotlin.md)。

---

## 核心概念（提纲）

- **Composable**：组合优于继承；纯函数倾向；**重组（Recomposition）** 范围最小化。
- **状态**：`remember`、`mutableStateOf`；状态提升；单向数据流与 **ViewModel** 配合。
- **副作用**：`LaunchedEffect`、`DisposableEffect`、`SideEffect`；避免组合体内非幂等操作。
- **导航**：`Navigation Compose`、Deep Link、类型安全路由（依版本）。

---

## 与 Android 互操作

- **ComposeView** 嵌入 XML；**AndroidView** 包装旧 View（相机预览等）。
- **WindowInsets**、软键盘、`WindowCompat`；与 **Edge-to-Edge** 全屏。

---

## Material / Material3

- **Theme**、`ColorScheme`、`Typography`、`Shape`；动态取色（Android 12+）。
- **组件**：Button、TextField、LazyColumn（虚拟列表）、Dialog、Snackbar。

---

## 性能（提纲）

- **`derivedStateOf`**、避免不稳定 lambda、`inline`/`key`、`Lazy*` prefetch。
- **调试**：Layout Inspector、`Recomposition counts`。

---

## 跨平台（Compose Multiplatform）

- **expect/actual**、分层模块；与 [KMP.md](../KMP/KMP.md) 交叉。
- **资源与生命周期**：依目标平台差异大，单独笔记。

---

## 延伸阅读

- [Kotlin.md](../../../Language/Kotlin.md)（协程、`StateFlow`）
- [Android.md](../../Android.md)
