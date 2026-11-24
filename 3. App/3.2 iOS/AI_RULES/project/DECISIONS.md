# 架构决策记录 (DECISIONS)

规范编号：PROJECT-002  
版本：1.0.0  
适用范围：强制执行  

---

## ADR-001: 采用 SwiftUI + Observation 框架作为唯一 UI 与状态响应方案

- **决策状态**：已确认 (Confirmed)
- **背景背景**：传统 UIKit 模版代码冗长，且旧版 `ObservableObject` + `@Published` 会导致全 View 无差别重绘，影响列表与复杂页面性能。
- **决策结论**：全工程统一采用 iOS 17+ 官方核心推荐的 **`@Observable` 宏**作为状态管理方案，ViewModel 统一绑定 `@MainActor`。
- **强制规则**：
  - 严禁手写旧版 `ObservableObject` 与 `@Published` 属性。
  - 视图只读取状态，更新状态必须调用 ViewModel 暴露的方法。

---

## ADR-002: 采用 `NavigationStack` + `NavigationPath` 强类型路由

- **决策状态**：已确认 (Confirmed)
- **背景背景**：已废弃的 `NavigationView` 与字符串 URL 跳转缺乏编译期校验，容易发生运行时缺失参数崩溃。
- **决策结论**：统一采用官方 **`NavigationStack` 配合 `NavigationPath`**，通过遵循 `Hashable` 与 `Codable` 的 `AppRoute` 枚举进行页面分发。
- **强制规则**：
  - 严禁裸写魔术字符串进行内部跳转。
  - 每个底导 Tab 维护独立的导航栈。

---

## ADR-003: 采用请求对象模式 (`ApiRequest`) 与统一响应外壳解包

- **决策状态**：已确认 (Confirmed)
- **背景背景**：分散拼装 `URLRequest` 会导致 Header 注入、鉴权 Token 刷新与错误处理逻辑严重重复。
- **决策结论**：网络请求统一封装为遵循 `ApiRequest` 协议的独立强类型结构体；由 `ApiClient.send()` 统一执行调度并自动剥离 `{ code, message, data }` 外层包装。
- **强制规则**：
  - 底层 `URLError` 统一转化为 `AppException` 密封枚举。
  - 业务 Model 仅反序列化内层 `data`，不侵入外层包装。

---

## ADR-004: 采用按 Feature 隔离的 String Catalogs (`.xcstrings`) 国际化体系

- **决策状态**：已确认 (Confirmed)
- **背景背景**：单个单体 `Localizable.strings` 极易引发高频 Git 合并冲突，且缺少强类型校验。
- **决策结论**：每个 Feature 拥有独立的 `.xcstrings` 文件，View 层通过类型安全的辅助扩展进行读取。

---

## ADR-005: 采用 Apple Unified Logging (`os.Logger`)

- **决策状态**：已确认 (Confirmed)
- **决策结论**：全面基于 `os.Logger`，按 `subsystem` 与 `category` 分类管理；严禁在代码中使用裸 `print()`。
