# KMP

> **Kotlin Multiplatform**：共享业务逻辑，多端 UI 分离（Android **Compose** / **XML**，iOS SwiftUI，Desktop，Web Wasm 等）。Compose Multiplatform 见 [JetpackCompose.md](../JetpackCompose/JetpackCompose.md)。语言层见 [Kotlin.md](../../../Language/Kotlin.md)。

---

## 分层思路（提纲）

- **commonMain**：expect 声明、共享数据模型、用例。
- **androidMain / iosMain / …**：actual 实现、平台 SDK。
- **依赖注入**：Koin / KMP 友好 DI 方案选型笔记。

---

## expect / actual

- **典型**：文件路径、加密、`SqlDriver`（SQLDelight）、`HttpClient`（Ktor）。
- **API 表面**：保持 **窄接口**，避免泄露平台类型到 common。

---

## 构建：Gradle KMP 插件

- **Targets**：`android`、`iosX64`/`iosArm64`、`iosSimulatorArm64` 等。
- **Hierarchy**：`apple`、`ios` 共享源码集。
- **CInterop**：与 Native（Objective-C）桥（按需）。

---

## 网络与存储

- **Ktor / OkHttp**（Android）与 NSURLSession（iOS）等在 actual 中封装。
- **SQLDelight**、Realm KMP 版本选型——随生态更新。

---

## 测试

- **commonTest**：`kotlin-test`；平台测试 `androidTest` / `iosTest`。

---

## 延伸阅读

- [Kotlin.md](../../../Language/Kotlin.md)
- [Android.md](../../Android.md)
