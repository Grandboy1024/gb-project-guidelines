# 架构决策记录 (DECISIONS)

规范编号：PROJECT-002  
版本：1.0.0  
适用范围：强制执行  

---

## ADR-001: 最低部署版本统一为 iOS 16.0+，采用 SwiftUI + `ObservableObject` 作为 UI 与状态响应方案

- **决策状态**：已确认 (Confirmed)
- **背景**：项目需面向更广泛的存量用户设备，最低部署目标统一为 **iOS 16.0+**。由于 `@Observable`、`SwiftData` 与 `ContentUnavailableView` 均为 iOS 17.0+ 专属 API，在 iOS 16.0+ 基线下直接使用会导致无法编译或运行时崩溃。
- **决策结论**：全工程统一以 **iOS 16.0+** 为最低支持版本，采用原生 **`ObservableObject` + `@Published`**（配合 `@StateObject` / `@ObservedObject` / `@EnvironmentObject`）与 **`CoreData`**，ViewModel 统一绑定 `@MainActor`，通过子视图拆分控制重绘粒度。
- **强制规则**：
  - 严禁使用 `@Observable`、`@Bindable`、`SwiftData`、`ContentUnavailableView` 等 iOS 17+ 专属 API。
  - 页面级 ViewModel 必须使用 `@StateObject` 持有，更新状态必须调用 ViewModel 暴露的方法。

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

---

## ADR-006: 统一采用 Swift Package Manager (SPM) 管理依赖，禁止引入 CocoaPods

- **决策状态**：已确认 (Confirmed)
- **背景**：CocoaPods 强依赖本机 Ruby 环境、`Podfile` 与 `.xcworkspace` 侵入性工程改造，且官方已进入维护与退场阶段；部分老旧 Pod 未适配 Swift 6 严格并发及 `PrivacyInfo.xcprivacy` 隐私清单，易引入协作冲突与审核风险。
- **决策结论**：全工程坚持 Apple 原生能力优先、极简依赖原则；确需引入经评审批准的第三方库（如 MQTT、崩溃监控等）时，统一使用 Xcode 原生 **Swift Package Manager (`SPM`)** 配合 `Package.resolved` 锁定版本；禁止新工程引入 CocoaPods。
- **强制规则**：
  - 严禁在工程中初始化或引入 `Podfile` / CocoaPods。
  - 若外部闭源硬件或商业 SDK 暂不支持 SPM，仅允许在评审备案后通过本地 `XCFramework` 方式接入。
