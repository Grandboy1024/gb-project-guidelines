# Swift & SwiftUI 编码与防御规范 (CODING)

规范编号：DEV-001  
版本：1.0.0  
适用范围：强制执行  

---

## 1. Modern Swift 语法与并发规范

1. **严格并发检查（Strict Concurrency Checking: Complete）**：
   - 必须通过 Swift 6 兼容的严格并发检查。跨并发隔离域传递的所有数据类型必须遵循 `Sendable` 协议。
   - UI 相关的属性与 ViewModel 类必须显式标注 `@MainActor`。
2. **穷尽模式匹配与 Switch 表达式**：
   - 针对密封枚举（如 `AppException`、`AsyncPhase`），必须使用 switch 进行穷尽匹配，严禁写出脆弱的 `default` 忽略关键错误分支：
     ```swift
     let message = switch exception {
     case .networkError(let msg): msg
     case .timeout: String(localized: "error_timeout", table: "Errors")
     case .unauthorized: String(localized: "error_unauthorized", table: "Errors")
     case .serverError(_, let msg), .businessError(_, let msg): msg
     case .invalidURL, .parseError, .cancelled, .forbidden, .unknown:
         String(localized: "error_unknown", table: "Errors")
     }
     ```
3. **不可变值类型优先（Immutability First）**：
   - 业务数据模型、请求参数、状态快照必须使用 `struct` 而非 `class`，除特殊需要外全部声明为 `let`。

---

## 2. 命名与文件组织约定

| 实体类别 | 命名风格 | 示范案例 |
|---|---|---|
| **文件名称** | `PascalCase.swift` | `LoginView.swift`、`AuthRepository.swift`、`LoginRequest.swift` |
| **结构体 / 类 / 枚举 / 协议** | `PascalCase` | `LoginView`、`AuthRepositoryProtocol`、`HttpMethod` |
| **方法 / 属性 / 参数** | `camelCase` | `fetchDevices()`、`deviceId`、`isLoading` |
| **私有成员** | `private` 修饰符 | `private var currentTask: Task<Void, Never>?` |
| **常量与设计令牌** | `lowerCamelCase` | `AppSpacing.md`、`AppColors.brandPrimary` |
| **强类型路由枚举项** | `camelCase` | `case deviceDetail(deviceId: String)` |

---

## 3. UI 边界条件与极端防御准则

1. **长文本与截断防护**：所有展示服务端动态文本的 `Text` 必须显式声明 `.lineLimit(...)` 与 `.truncationMode(.tail)`，严禁撑破卡片或将核心操作挤出屏幕。
2. **触控热区保证（44 × 44 pt）**：所有可交互图标按钮，必须通过 `.frame(width: 44, height: 44)` 确保触控响应区域满足 Apple HIG 规范。
3. **软键盘防遮挡与滚动交互**：表单界面必须由 `ScrollView` 包裹，并配置 `.scrollDismissesKeyboard(.immediately)`。
4. **加载中按钮防重点击**：网络请求处理期间，操作按钮必须自动进入 `isLoading` 状态并处于 `disabled` 状态，彻底杜绝重复提交。
5. **四态闭环**：页面必须闭环处理「空状态 (Empty)、加载中 (Loading)、异常 (Error 带重试)、成功 (Success)」。
