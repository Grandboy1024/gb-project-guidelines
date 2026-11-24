# Kotlin & Compose 编码与防御规范 (CODING)

规范编号：DEV-001  
版本：1.0.0  
适用范围：强制执行  

---

## 1. Modern Kotlin 2.x 语法规范

1. **穷尽模式匹配与 when 表达式**：
   - 针对密封类层级（如 `AppException`、`UiAction`），必须使用 when 表达式进行穷尽匹配，严禁遗漏分支或写出脆弱的 `else` 忽视关键分支：
     ```kotlin
     val message = when (exception) {
         is AppException.NetworkError -> exception.detail
         is AppException.Timeout -> stringResource(R.string.error_timeout)
         is AppException.Unauthorized -> stringResource(R.string.error_unauthorized)
         is AppException.ServerError -> exception.detail
         is AppException.BusinessError -> exception.detail
         is AppException.InvalidUrl, is AppException.ParseError, is AppException.Cancelled, is AppException.Forbidden, is AppException.Unknown ->
             stringResource(R.string.error_unknown)
     }
     ```
2. **不可变数据模型优先（Immutability First）**：
   - 业务数据模型、请求参数、状态快照必须使用 `data class`，字段一律声明为只读 `val`。
   - 状态类必须添加 `@Immutable` 或 `@Stable` 注解以保证 Compose 重组跳过效率。

---

## 2. 命名与文件组织约定

| 实体类别 | 命名风格 | 示范案例 |
|---|---|---|
| **文件名称** | `PascalCase.kt` | `LoginScreen.kt`、`AuthRepository.kt`、`LoginRequest.kt` |
| **类 / 接口 / 枚举 / 对象** | `PascalCase` | `LoginScreen`、`AuthRepository`、`HttpMethod` |
| **函数 / 方法 / 变量** | `camelCase` | `fetchDevices()`、`deviceId`、`isLoading` |
| **Composable 界面函数** | `PascalCase` | `LoginScreen()`、`DeviceCard()` |
| **常量与设计令牌** | `lowerCamelCase` | `AppSpacing.md`、`AppRadius.card` |
| **强类型路由类** | `PascalCase` | `data object Login : AppRoute`、`data class DeviceDetail(...)` |

---

## 3. UI 边界条件与极端防御准则

1. **长文本与截断防护**：所有展示服务端动态文本的 `Text` 必须显式声明 `maxLines = 1` 与 `overflow = TextOverflow.Ellipsis`，严禁撑破卡片或将核心操作挤出屏幕。
2. **触控热区保证（48 × 48 dp）**：所有可交互图标按钮，必须通过 `Modifier.size(48.dp)` 确保触控响应区域满足 Android 无障碍规范。
3. **软键盘防遮挡与滚动交互**：表单界面必须由 `Modifier.verticalScroll(...)` 与 `Modifier.imePadding()` 双重保护。
4. **加载中按钮防重点击**：网络请求处理期间，操作按钮必须自动进入 `isLoading` 状态并在 `enabled = false` 下被禁用，彻底杜绝重复提交。
5. **四态闭环**：页面必须闭环处理「空状态 (Empty)、加载中 (Loading)、异常 (Error 带重试)、成功 (Success)」。
