# 架构分层与工程设计规范 (ARCHITECTURE)

规范编号：DEV-002  
版本：1.0.0  
适用范围：强制执行  

---

## 1. 架构总蓝图：Clean MVVM (SwiftUI)

项目全面落地以 Clean MVVM 为核心的架构体系，确保单向数据流与清晰的职责边界。

```mermaid
flowchart LR
    View["视图层 View<br/>(SwiftUI Struct)"] -->|"转发用户意图"| VM["控制器 ViewModel<br/>(ObservableObject & @MainActor)"]
    VM -->|"调用仓储接口"| Repo["仓储层 Repository<br/>(数据唯一网关)"]
    Repo -->|"发送强类型请求"| Api["网络客户端 ApiClient / URLSession"]
    Repo -->|"读写加密凭证/缓存"| Storage["Keychain / CoreData"]
    Repo -.->|"返回不可变 Model"| VM
    VM -.->|"暴露 @Published / AsyncPhase"| View
```

### 1.1 核心分层职责矩阵

| 分层 | 典型实现 | 核心职责 | 绝对严禁的行为 (Anti-Patterns) |
|---|---|---|---|
| **View** | SwiftUI `struct: View` | 依据 ViewModel 暴露的只读状态渲染 UI；捕获手势并转发为意图调用；处理弹窗与转场。 | 严禁编写任何业务逻辑；严禁直接调用 Repository 或 ApiClient；严禁私自修改状态；严禁使用 iOS 17+ 专属 API。 |
| **ViewModel** | `@MainActor final class: ObservableObject` | 持有 `@Published private(set)` UI 状态；调度结构化异步并发 `Task`；调用 Repository 获取数据并捕获 `AppException`。 | 严禁 `import SwiftUI`；严禁持有 `UIView` 或任何视图实例；严禁省略 `@MainActor`；严禁使用 `@Observable`。 |
| **Repository** | `XxxRepository: XxxRepositoryProtocol` | ViewModel 获取数据的唯一入口；拦截底层 `URLError` / `DecodingError` 并统一转换为 `AppException`。 | 严禁向上层抛出未处理的底层原始网络异常；严禁将未解码的裸 Data 暴露给 ViewModel。 |
| **Request Object** | 遵循 `ApiRequest` 协议的不可变 `struct` | 封装具体 API 的 Path、Method、Headers、Query、Body 及其专有泛型返回类型 `Response`。 | 严禁在业务代码中散落直接拼装 `URLRequest`。 |

---

## 2. 单向依赖流动铁律

```
View → ViewModel → Repository → ApiClient / LocalStorage
```

- **依赖单向不可逆**：上层可以依赖下层，下层绝对禁止引用上层。
- **Feature 互相物理隔离**：**Feature A 严禁直接 import 或依赖 Feature B 的私有代码**。
- **跨模块共享机制**：多模块共用的公共逻辑下沉至 `Core/` 或 `Shared/`；跨模块读取业务数据统一通过全局服务或在 App 容器中装配。

---

## 3. 状态管理与生命周期 (ObservableObject, iOS 16.0+)

1. **ViewModel 即状态中枢**：
   - 每个页面的业务状态统一封装在继承 `ObservableObject` 并标记 `@MainActor` 的 ViewModel 类中。
   - 简单异步提交操作使用 `AsyncPhase<Void>`；数据展示型列表使用 `AsyncPhase<[T]>`。
2. **生命周期绑定**：
   - 页面级 ViewModel 由 SwiftUI 视图通过 `@StateObject` 持有，页面退出时通过 `.onDisappear` 或 `.task` 取消进行中的 `Task`。
   - 全局会话状态（如 `AppSession`、`AppRouter`）由 App 根级通过 `@EnvironmentObject` 向外分发。
