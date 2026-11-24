# 架构决策记录 (DECISIONS)

规范编号：PROJECT-002  
版本：1.0.0  
适用范围：强制执行  

---

## ADR-001: 采用 Jetpack Compose + UDF 作为唯一 UI 与状态响应方案

- **决策状态**：已确认 (Confirmed)
- **背景背景**：传统 XML 布局层级冗余、双向绑定状态易错，难以维护一致的数据流。
- **决策结论**：全工程统一采用 **Jetpack Compose + Material 3**，状态管理采用 `StateFlow` + UDF（单向数据流），ViewModel 负责状态下发与事件处理。
- **强制规则**：
  - 严禁手写 XML 布局文件。
  - Composable 视图层严禁直接发起网络调用。

---

## ADR-002: 采用 Navigation Compose 2.8+ 强类型路由

- **决策状态**：已确认 (Confirmed)
- **背景背景**：旧版字符串路由拼接极易在运行时发生参数缺失或类型转换崩溃。
- **决策结论**：统一采用官方 **Navigation Compose 2.8+ 强类型路由**，通过标记 `@Serializable` 的类与对象进行导航传参。
- **强制规则**：
  - 严禁裸写魔术字符串进行页面跳转。
  - 单 Activity（`MainActivity`）全 Compose 架构。

---

## ADR-003: 采用请求对象模式 (`ApiRequest`) 与统一响应外壳解包

- **决策状态**：已确认 (Confirmed)
- **背景背景**：分散手写 Retrofit 接口会导致 Header 附加、Token 刷新与错误处理逻辑严重重复。
- **决策结论**：网络请求统一封装为遵循 `ApiRequest<T>` 接口的强类型类；由 `ApiClient.send()` 统一执行调度并自动剥离 `{ code, message, data }` 外层包装。
- **强制规则**：
  - 底层网络异常统一转化为 `AppException` 密封类。
  - 业务 Model 仅反序列化内层 `data`。

---

## ADR-004: 采用 Hilt 作为唯一依赖注入框架

- **决策状态**：已确认 (Confirmed)
- **决策结论**：全工程统一使用 Google 官方推荐的 **Hilt** 接管依赖注入与生命周期管束。
- **强制规则**：
  - ViewModel 必须标注 `@HiltViewModel` 与 `@Inject`。
  - 严禁手动创建单例或手写 Service Locator。

---

## ADR-005: 采用 Timber 作为唯一日志门面

- **决策状态**：已确认 (Confirmed)
- **决策结论**：全面基于 `Timber`；严禁在业务代码中使用原生 `Log.d()` 与 `println()`。
