# 规范变更记录 (CHANGELOG)

所有 AI 规范体系的升级、废止与重大决策修订均记录于此。

---

## [1.1.0] - 2026-11-25

### 变更 (Changed)
- 统一最低部署版本为 **iOS 16.0+**，状态管理由 iOS 17+ `@Observable` 调整为 iOS 16 兼容的 `ObservableObject` + `@Published` + `@StateObject` / `@EnvironmentObject`，结构化本地缓存明确采用 `CoreData`，替换 `ContentUnavailableView` 为 `AppEmptyView`。
- 对齐《RESTful API 规范》与《统一错误码与异常处理规范》：修正 `ApiResponseEnvelope` 成功码为 `0`、补充 `traceId` / `isRetryable` / `EmptyResponse`、修复 `Encodable` Body 序列化与 RFC 3339 毫秒日期解析、新增 `NetworkInterceptor` 与 `actor AuthTokenCoordinator` 并发刷新范式。
- 修正 Dynamic Type 语义字阶、路由穷尽匹配与 Sheet/FullScreenCover 绑定、`Swift Testing` 消除 `Task.sleep` 竞态，并补充 `SwiftLint` / `PrivacyInfo.xcprivacy` / `Core/BLE` 规范。
- 新增 `ADR-006`：明确依赖包管理统一采用原生 **SPM**，禁止引入 CocoaPods，无 SPM 支持的闭源 SDK 仅允许以本地 `XCFramework` 评审接入。

## [1.0.0] - 2026-11-24

### 新增 (Added)
- 初始化 iOS 原生移动端应用 AI 规范体系 (`AI_RULES`)。
- 制定 AI 核心行为准则与十条绝对红线 (`core/MASTER.md`)。
- 确立标准五步法作业流程 SOP (`core/WORKFLOW.md`) 与沟通汇报规范 (`core/COMMUNICATION.md`)。
- 确立 Swift 5.9+ / 6 严格并发与现代编码规范 (`development/CODING.md`)。
- 确立 Clean MVVM、SwiftUI Observation 细粒度状态管理、NavigationStack 强类型路由与请求对象网络层架构 (`development/ARCHITECTURE.md`)。
- 制定 Swift Testing 自动化测试与质量保障体系 (`development/TESTING.md`)。
- 固化业务背景 (`project/CONTEXT.md`)、八项核心架构决策 ADR-001 ~ ADR-008 (`project/DECISIONS.md`) 与项目就绪状态 (`project/STATUS.md`)。
