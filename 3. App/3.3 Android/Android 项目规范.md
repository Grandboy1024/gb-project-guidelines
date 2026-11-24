## 一、技术选型

| 类别 | 选择 | 说明 |
| --- | --- | --- |
| 语言与编译器 | **Kotlin 2.0+ (K2 编译器)** | 开启显式 API 模式与严格类型安全，全面采用现代 Kotlin 特性（Coroutines 携程、Flow 响应流、Sealed Interface 密封接口、Data Object） |
| 架构模式 | **Clean MVVM / MVI** | 见第三节，View (Jetpack Compose) / ViewModel (`androidx.lifecycle.ViewModel`) / Repository / ApiClient / LocalDataSource |
| UI 框架 | **Jetpack Compose + Material 3** | 全声明式 UI，状态下沉与事件上报（UDF），组件原子化解耦，原生支持 Edge-to-Edge 全面屏与动态色彩 |
| 状态管理 | **`StateFlow<UiState<T>>` + UDF 单向数据流** | 配合 `collectAsStateWithLifecycle()` 实现生命周期感知型细粒度渲染，杜绝无意义重组（Recomposition） |
| 依赖注入 | **Hilt (`dagger.hilt.android`)** | Google 官方推荐标准方案，统一接管 Application、ViewModel、Repository 及网络单例装配 |
| 路由导航 | **Navigation Compose 2.8+ 强类型路由** | 见第五节，基于 `@Serializable` 路由对象与类，彻底告别旧版魔术字符串拼接传参 |
| 网络层 | **OkHttp 4.x + Retrofit 2.x + 请求对象模式 (`ApiRequest<T>`)** | 见第六节，纯协程挂起函数调度，内置统一响应体外壳解包、Token 自动刷新与 CoroutineScope 生命周期取消 |
| 数据类 / 序列化 | **`kotlinx.serialization`** | 官方编译器插件，无反射高性能，模型全部基于不可变不可变 `data class` |
| 本地存储 | **Jetpack `DataStore`**（Preferences/Proto）+ **`EncryptedSharedPreferences`**（Token）+ **`Room`**（本地缓存） | 严禁明文持久化 Token 与敏感机密 |
| 错误处理 | **自定义 `AppException` 密封类层级 + `UiState<T>`** | 见第六节与第四节，强类型穷尽错误捕获，不向上层暴露底层 `IOException` / `HttpException` |
| 国际化 | **Android Resources (`strings.xml`)**，**按 feature 隔离** | 见第七节，支持多模块资源隔离与统一包装访问，杜绝硬编码 |
| UI 规范 / 主题 / 间距 | **Material 3 + `AppTheme` (Design Tokens)** | 见第八节，定义 `AppSpacing`、`AppRadius`、`AppColors` 语义色、48dp 触控热区、统一排版字阶与防御性编程 |
| 环境配置 | **Gradle BuildVariants + BuildConfig** | 见第九节，支持 `debug`、`staging`、`release` 集中环境动态编译切换 |
| 日志系统 | **`Timber`** | Debug 模式植入 `DebugTree`，Release 模式挂载 Crashlytics / 异常上报，严禁使用 `Log.d` / `println` |
| 测试框架 | **`JUnit 5` + `MockK` + `kotlinx-coroutines-test` + `Turbine`** | 见第九节，覆盖 Repository 协程测试、ViewModel 状态流测试与 Compose 交互测试 |
| 代码规范 / Lint | **`ktlint` + `detekt` + Android Lint** | 见第九节，初期开启最严格静态代码检查规则与 CI 门禁强校验 |

---

## 二、目录结构

```
my-android-app/
├── AI_RULES/                     # AI 协作主规范体系（单一真实来源，所有 AI 工具共用）
│   ├── RULES.md                  # 规范总入口 / 规范清单 (Manifest)
│   ├── core/                     # 核心规范：MASTER(行为), WORKFLOW(流程), COMMUNICATION(沟通)
│   ├── development/              # 开发规范：CODING(编码), ARCHITECTURE(架构), TESTING(测试)
│   ├── project/                  # 项目知识：CONTEXT(背景), DECISIONS(技术决策), STATUS(进度)
│   └── CHANGELOG.md              # 规范变更记录
├── AGENTS.md                     # Codex / Copilot / OpenDevin 平台适配入口
├── CLAUDE.md                     # Claude Code 平台适配入口
├── GEMINI.md                     # Gemini CLI / Antigravity 平台适配入口
├── gradle/
│   └── libs.versions.toml        # Gradle Version Catalog 依赖版本集中管理
├── app/
│   ├── build.gradle.kts          # 模块 Gradle 配置
│   └── src/
│       ├── main/
│       │   ├── AndroidManifest.xml # 应用清单配置
│       │   ├── java/com/fuxincanrui/app/
│       │   │   ├── App.kt        # Application 类（@HiltAndroidApp，初始化日志、存储）
│       │   │   ├── MainActivity.kt # 单 Activity 架构宿主（ComponentActivity，Edge-to-Edge）
│       │   │   ├── navigation/   # 强类型路由配置
│       │   │   │   ├── AppNavHost.kt # Navigation Compose 宿主容器
│       │   │   │   ├── AppRoute.kt   # @Serializable 路由对象定义
│       │   │   │   └── AppNavigator.kt # 统一导航中枢
│       │   │   ├── theme/        # Material 3 主题、色彩、字体与间距令牌
│       │   │   │   ├── Color.kt      # 语义色彩体系（支持浅色/深色自适应）
│       │   │   │   ├── Spacing.kt    # 4dp/8dp 网格间距体系 (AppSpacing)
│       │   │   │   ├── Shape.kt      # 圆角常量系统 (AppRadius)
│       │   │   │   ├── Type.kt       # 统一排版与字体阶梯 (Typography)
│       │   │   │   └── Theme.kt      # AppTheme 组合提供者
│       │   │   ├── core/         # 与业务完全解耦的基础设施
│       │   │   │   ├── config/   # 环境配置（BuildConfig 映射）
│       │   │   │   ├── network/  # 网络层基建
│       │   │   │   │   ├── ApiClient.kt      # OkHttp + Retrofit 调度中枢
│       │   │   │   │   ├── ApiRequest.kt     # 请求基类抽象
│       │   │   │   │   ├── interceptors/     # 统一头、鉴权、日志拦截器
│       │   │   │   │   └── envelope/         # 通用响应外壳模型与解包器
│       │   │   │   ├── storage/  # 本地持久化封装
│       │   │   │   │   ├── DataStoreManager.kt
│       │   │   │   │   └── KeychainStorage.kt # 加密凭证存储
│       │   │   │   ├── error/
│       │   │   │   │   └── AppException.kt   # 统一密封类错误体系
│       │   │   │   ├── logging/
│       │   │   │   │   └── AppLogger.kt      # Timber 门面包装
│       │   │   │   └── di/       # 全局 Hilt 依赖注入模块 (SingletonComponent)
│       │   │   ├── shared/       # 跨 Feature 高度复用的通用模块
│       │   │   │   ├── components/ # 通用基础 UI 组件（AppButton、AppTextField、AsyncPhaseView）
│       │   │   │   ├── extensions/ # 协程与 Compose 扩展
│       │   │   │   └── model/      # 全局通用不可变模型
│       │   │   └── features/     # 业务模块目录（完全高内聚、物理隔离）
│       │   │       └── auth/     # 认证模块 Feature
│       │   │           ├── data/ # 数据层（本 Feature 内部共享）
│       │   │           │   ├── requests/ # LoginRequest 等强类型请求对象
│       │   │           │   ├── models/   # @Serializable 不可变数据模型 (User, Token)
│       │   │           │   ├── repository/ # AuthRepository 接口与实现
│       │   │           │   └── di/       # 本 Feature 专用的 Hilt Module
│       │   │           ├── AuthSession.kt # 跨页面共享的全局认证会话
│       │   │           ├── login/    # 登录页面就近聚合
│       │   │           │   ├── LoginScreen.kt    # Compose 视图组件
│       │   │           │   ├── LoginViewModel.kt # 基于 ViewModel + StateFlow
│       │   │           │   ├── LoginUiState.kt   # 登录状态数据类
│       │   │           │   └── components/       # 登录页私有子组件
│       │   │           └── register/ # 注册页面
│       │   │               ├── RegisterScreen.kt
│       │   │               └── RegisterViewModel.kt
│       │   └── res/
│       │       ├── values/
│       │       │   ├── strings.xml     # 英文/通用基础文案
│       │       │   └── strings_auth.xml # Feature 隔离的认证文案
│       │       └── values-zh-rCN/
│       │           ├── strings.xml     # 简体中文文案
│       │           └── strings_auth.xml
│       └── test/                     # 单元测试工程（镜像对应 main/java/）
│           └── java/com/fuxincanrui/app/
│               └── features/auth/
│                   ├── LoginViewModelTest.kt
│                   └── AuthRepositoryTest.kt
```

### 目录组织规则

1. **就近聚合（Colocation），消除深层冗余**：摒弃传统庞大的 `fragment/`、`presenter/` 跨越式分包。页面直接在 Feature 下按功能建目录（如 `login/`、`register/`）。一个页面的 View（`LoginScreen.kt`）、ViewModel（`LoginViewModel.kt`）和状态契约（`LoginUiState.kt`）聚合在一起，降低目录嵌套深度，修改页面无需在深层目录树中上下跳跃。
2. **UDF 单向数据流与不可变状态**：ViewModel 暴露不可变 `StateFlow<UiState>`，UI 仅负责将状态声明式渲染上屏并通过 Lambda 向上回传用户事件（`UiAction`），杜绝双向篡改。
3. **数据层保持在 Feature 级**：`data/`（Repository、Requests、Models）由整个 Feature 共享，不按单一页面下沉。模型采用遵循 `kotlinx.serialization` 的不可变 `data class`，集中在 `data/` 能保证边界清晰并杜绝不同 Feature 间互相破坏性耦合。
4. **生命周期感知与自动释放**：
   - 页面级 ViewModel 通过 `@HiltViewModel` 声明并依托 Compose 导航回退栈（NavBackStackEntry）生命周期自动销毁与释放协程；
   - 跨页面常驻的会话级状态（如 `AuthSession`），注入在 SingletonComponent 中，保证全局唯一真实来源。
5. **组件归宿清晰明确**：
   - 仅单个页面私有的组件放该页面目录下的 `components/`；
   - 本 Feature 内两个以上页面共用的组件放 `features/<feature>/components/`；
   - 跨两个以上 Feature 共用的通用原子组件必须提升到根目录 `shared/components/`。
6. **国际化资源隔离**：通过 `res/values/strings_<feature>.xml` 建立模块化多语言资源，通过强类型访问工具或标准 R 文件引用，严禁跨 Feature 乱串文案。
7. **强类型导航与参数安全**：基于 Navigation Compose 2.8+ 体系，所有路由统一声明为 `@Serializable` 的 `data object` 或 `data class`，杜绝旧版手写动态拼接字符串引发的运行时奔溃。
8. **设计系统与硬编码防御**：组件间距一律使用 `AppSpacing` 常量；字体、圆角与颜色必须通过 `MaterialTheme` 与语义色获取，严禁在 Composable 中硬编码诸如 `Color(0xFF123456)` 或固定数值 Padding。
9. **测试目录与源码镜像**：测试文件命名以 `Test.kt` 结尾，目录结构与被测源码保持 1:1 镜像对齐。
10. **AI 规范单一真实来源**：根目录下的 `AI_RULES/` 为唯一真理源，`AGENTS.md`、`CLAUDE.md`、`GEMINI.md` 仅作为各平台的软链接入口。

---

## 三、MVVM / MVI 落地规则

### 3.1 角色对应

| 角色 | 典型实现 | 职责 |
| --- | --- | --- |
| **View** | `@Composable fun XxxScreen(...)` | 依据 UiState 声明式渲染 UI，转发用户手势事件，响应一次性效果（Side Effects），绝不含复杂业务逻辑与网络代码 |
| **ViewModel** | `@HiltViewModel class XxxViewModel : ViewModel()` | 持有单一不可变 `StateFlow<UiState>`，调度协程 `viewModelScope.launch`，调用 Repository，向 UI 暴露只读流 |
| **Model** | Repository + `data class Model` | Repository 是 ViewModel 获取数据的唯一网关，屏蔽底层网络、缓存与 Room 存储细节 |

**依赖方向（单向流动）**：

```
View (Compose) → ViewModel (StateFlow) → Repository → ApiClient / LocalStorage
```

禁止反向引用；View 严禁直接绕过 ViewModel 调用 Repository 或 ApiClient。

---

### 3.2 示例：登录

#### 1. UiState 与 UiAction 定义

```kotlin
// features/auth/login/LoginUiState.kt
package com.fuxincanrui.app.features.auth.login

import androidx.compose.runtime.Immutable
import com.fuxincanrui.app.core.error.AppException

@Immutable
data class LoginUiState(
    val email: String = "",
    val password: String = "",
    val isLoading: Boolean = false,
    val errorMessage: String? = null,
    val isSuccess: Boolean = false
) {
    val isInputValid: Boolean
        get() = email.isNotBlank() && password.length >= 6
}

sealed interface LoginUiAction {
    data class EmailChanged(val value: String) : LoginUiAction
    data class PasswordChanged(val value: String) : LoginUiAction
    data object Submit : LoginUiAction
    data object ClearError : LoginUiAction
}
```

#### 2. ViewModel（基于 Hilt 与协程）

```kotlin
// features/auth/login/LoginViewModel.kt
package com.fuxincanrui.app.features.auth.login

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.fuxincanrui.app.core.error.AppException
import com.fuxincanrui.app.features.auth.data.repository.AuthRepository
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.Job
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.launch
import javax.inject.Inject

@HiltViewModel
class LoginViewModel @Inject constructor(
    private val authRepository: AuthRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(LoginUiState())
    val uiState: StateFlow<LoginUiState> = _uiState.asStateFlow()

    private var loginJob: Job? = null

    fun onAction(action: LoginUiAction) {
        when (action) {
            is LoginUiAction.EmailChanged -> _uiState.update { it.copy(email = action.value) }
            is LoginUiAction.PasswordChanged -> _uiState.update { it.copy(password = action.value) }
            is LoginUiAction.ClearError -> _uiState.update { it.copy(errorMessage = null) }
            is LoginUiAction.Submit -> submit()
        }
    }

    private fun submit() {
        val currentState = _uiState.value
        if (!currentState.isInputValid || currentState.isLoading) return

        // 防重复提交：取消之前未完成的任务
        loginJob?.cancel()

        loginJob = viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, errorMessage = null) }

            try {
                authRepository.login(currentState.email, currentState.password)
                _uiState.update { it.copy(isLoading = false, isSuccess = true) }
            } catch (e: AppException) {
                _uiState.update { it.copy(isLoading = false, errorMessage = e.message) }
            } catch (e: Throwable) {
                _uiState.update { it.copy(isLoading = false, errorMessage = e.localizedMessage) }
            }
        }
    }
}
```

#### 3. Repository

```kotlin
// features/auth/data/repository/AuthRepository.kt
package com.fuxincanrui.app.features.auth.data.repository

import com.fuxincanrui.app.core.network.ApiClient
import com.fuxincanrui.app.features.auth.data.models.AuthToken
import com.fuxincanrui.app.features.auth.data.requests.LoginRequest
import com.fuxincanrui.app.core.storage.KeychainStorage
import javax.inject.Inject
import javax.inject.Singleton

interface AuthRepository {
    suspend fun login(email: String, password: String): AuthToken
}

@Singleton
class AuthRepositoryImpl @Inject constructor(
    private val apiClient: ApiClient,
    private val storage: KeychainStorage
) : AuthRepository {

    override suspend fun login(email: String, password: String): AuthToken {
        val request = LoginRequest(email = email, password = password)
        val token = apiClient.send(request)
        // 缓存安全凭证
        storage.saveAccessToken(token.accessToken)
        return token
    }
}
```

#### 4. View (Jetpack Compose)

```kotlin
// features/auth/login/LoginScreen.kt
package com.fuxincanrui.app.features.auth.login

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.verticalScroll
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalFocusManager
import androidx.compose.ui.res.stringResource
import androidx.compose.ui.text.input.ImeAction
import androidx.compose.ui.text.input.KeyboardType
import androidx.compose.ui.unit.dp
import androidx.hilt.navigation.compose.hiltViewModel
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import com.fuxincanrui.app.R
import com.fuxincanrui.app.shared.components.AppButton
import com.fuxincanrui.app.shared.components.AppTextField
import com.fuxincanrui.app.theme.AppSpacing

@Composable
fun LoginRoute(
    onNavigateToHome: () -> Unit,
    viewModel: LoginViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    LaunchedEffect(uiState.isSuccess) {
        if (uiState.isSuccess) {
            onNavigateToHome()
        }
    }

    LoginScreen(
        uiState = uiState,
        onAction = viewModel::onAction
    )
}

@Composable
fun LoginScreen(
    uiState: LoginUiState,
    onAction: (LoginUiAction) -> Unit
) {
    val focusManager = LocalFocusManager.current
    val scrollState = rememberScrollState()

    Column(
        modifier = Modifier
            .fillMaxSize()
            .background(MaterialTheme.colorScheme.background)
            .statusBarsPadding()
            .navigationBarsPadding()
            .imePadding()
            .verticalScroll(scrollState)
            .padding(horizontal = AppSpacing.screenMargin, vertical = AppSpacing.xl),
        verticalArrangement = Arrangement.Center
    ) {
        Text(
            text = stringResource(R.string.auth_login_title),
            style = MaterialTheme.typography.headlineLarge,
            color = MaterialTheme.colorScheme.onBackground
        )
        Spacer(modifier = Modifier.height(AppSpacing.xs))
        Text(
            text = stringResource(R.string.auth_login_subtitle),
            style = MaterialTheme.typography.bodyMedium,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )

        Spacer(modifier = Modifier.height(AppSpacing.lg))

        // 邮箱输入框
        AppTextField(
            value = uiState.email,
            onValueChange = { onAction(LoginUiAction.EmailChanged(it)) },
            placeholder = stringResource(R.string.auth_email_placeholder),
            keyboardType = KeyboardType.Email,
            imeAction = ImeAction.Next
        )

        Spacer(modifier = Modifier.height(AppSpacing.md))

        // 密码输入框
        AppTextField(
            value = uiState.password,
            onValueChange = { onAction(LoginUiAction.PasswordChanged(it)) },
            placeholder = stringResource(R.string.auth_password_placeholder),
            isPassword = true,
            imeAction = ImeAction.Done,
            onImeAction = {
                focusManager.clearFocus()
                if (uiState.isInputValid) {
                    onAction(LoginUiAction.Submit)
                }
            }
        )

        if (uiState.errorMessage != null) {
            Spacer(modifier = Modifier.height(AppSpacing.sm))
            Text(
                text = uiState.errorMessage,
                style = MaterialTheme.typography.bodySmall,
                color = MaterialTheme.colorScheme.error
            )
        }

        Spacer(modifier = Modifier.height(AppSpacing.xl))

        // 登录主操作按钮
        AppButton(
            text = stringResource(R.string.auth_login_button),
            isLoading = uiState.isLoading,
            enabled = uiState.isInputValid && !uiState.isLoading,
            onClick = {
                focusManager.clearFocus()
                onAction(LoginUiAction.Submit)
            }
        )
    }
}
```

---

### 3.3 十条硬性规则

1. **全面采用 `collectAsStateWithLifecycle()`**：在 Compose 视图层消费 `StateFlow` 时，必须使用 `collectAsStateWithLifecycle()`，禁止在后台停止阶段无效重绘，节约系统资源。
2. **ViewModel 必须持有单一不可变状态**：UI 状态必须封装为单一不可变 `UiState` 数据类，严禁在 ViewModel 中散落暴露数十个孤立的 `mutableStateOf`。
3. **数据模型不可变性（Immutability）**：Model 层一律使用只读 `data class` 并遵循不可变规则，严禁声明 `var` 属性。
4. **禁止在 ViewModel 中引入 Android Context / 视图依赖**：ViewModel 严禁持有 `Context`、`View`、`Activity` 或任何 Compose 视图实例，从根源切断内存泄漏。
5. **错误多语言延迟翻译**：ViewModel 和 Repository 仅透传强类型 `AppException` 密封类体系，多语言文案必须由 View 层在渲染时根据 `R.string` 进行匹配呈现。
6. **Repository 屏障防御**：Repository 必须拦截底层 `IOException`、`HttpException` 或解析异常，并转换为上层友好的 `AppException`，严禁让未处理的网络层原始异常穿透到 UI。
7. **`domain/` 目录按需引入**：默认不设立多余的 Domain/UseCase 胶水层，Repository 直接与 ViewModel 对接；仅当某一段业务逻辑在 3 个以上 ViewModel 间高频复用时才提取独立 UseCase。
8. **Feature 物理隔离**：Feature 之间严禁交叉引用彼此私有代码；全局共享逻辑必须下沉到 `core/` 或 `shared/`。
9. **协程取消与生命周期绑定**：后台任务必须通过 `viewModelScope` 派生调度，页面退出时自动取消任务；网络密集循环中必须配合 `ensureActive()` 防御。
10. **统一命名约定**：严格遵照 `XxxScreen.kt`、`XxxViewModel.kt`、`XxxRepository.kt`、`XxxRequest.kt` 规范命名，类名与文件名保持严格一致。

---

## 四、状态管理与 Compose 渲染规范

项目基于 **StateFlow + UDF（单向数据流）** 结合 Jetpack Compose 现代化重组机制构建统一响应式状态层。

### 4.1 核心原则与架构定位

1. **UDF（Unidirectional Data Flow，单向数据流）**：
   - 状态向下流动（State Flows Down）：ViewModel 计算出最新 `UiState` 并向下推送到 Composable。
   - 事件向上流动（Events Flow Up）：用户手势与意图通过回调函数（Lambda）向上通知 ViewModel。
2. **状态提升（State Hoisting）**：
   - 视图层将有状态组件（Stateful Composable，如负责与 ViewModel 绑定的 `LoginRoute`）与无状态组件（Stateless Composable，如纯渲染的 `LoginScreen`）明确拆分，极大提升预览能力与可测性。
3. **细粒度重组保护**：
   - 保证传递给子 Composable 的入参具有 `@Immutable` 或 `@Stable` 稳定性保证，防止不必要的重组引发掉帧。

### 4.2 异步状态高级模式（`UiState<T>` 范式）

针对通用的异步网络拉取界面，强烈推荐使用密封接口统一封装：

```kotlin
// core/network/UiState.kt
package com.fuxincanrui.app.core.network

import androidx.compose.runtime.Immutable
import com.fuxincanrui.app.core.error.AppException

@Immutable
sealed interface AsyncUiState<out T> {
    data object Idle : AsyncUiState<Nothing>
    data object Loading : AsyncUiState<Nothing>
    data class Success<T>(val data: T) : AsyncUiState<T>
    data class Error(val exception: AppException) : AsyncUiState<Nothing>
}
```

### 4.3 状态管理八条禁忌（Anti-Patterns）

1. **严禁在 Composable 内部随意声明 `rememberCoroutineScope` 发起业务网络请求**：业务请求必须在 ViewModel 中由 `viewModelScope` 统一发起与管理生命周期。
2. **严禁在 Compose 渲染逻辑中直接执行副作用**：改变状态、上报埋点或执行 I/O 必须放入 `LaunchedEffect` 或 `DisposableEffect` 中。
3. **严禁在 ViewModel 中使用 `MutableStateFlow.value = ...` 进行非原子更新**：多并发场景下必须使用 `.update { ... }` 保证线程安全状态跃迁。
4. **严禁将未标记稳定性的复杂集合裸传进叶子组件**：推荐使用不可变集合包装或标准不可变数据结构，避免破坏 Compose 重组跳过机制（Recomposition Skip）。
5. **严禁使用 EventBus / BroadcastReceiver 替代业务状态传递**：组件之间通信必须统一走 ViewModel 驱动的 StateFlow / SharedFlow。
6. **严禁忽视协程取消（CancellationException）**：在捕获异常时，若直接使用 `catch (e: Exception)` 必须显式判断 `if (e is CancellationException) throw e`，切勿吞噬协程取消指令。
7. **严禁将全局静态单例作为共享状态持有者**：除基础工具库外，业务状态严禁存储在静态单例中，必须通过 Hilt 统一接管生命周期。
8. **严禁在 Composable 方法体内部进行繁重的集合遍历排序**：数据处理一律在 ViewModel 协程流中通过 `map`、`filter` 计算完毕后再交付给 UI 渲染。

---

## 五、路由规范（Navigation Compose 2.8+ 强类型路由）

项目全面摒弃旧版字符串 URL 拼接路由，全量采用 Navigation Compose 2.8+ 官方强类型路由方案（Type-Safe Navigation）。

### 5.1 核心设计与优势

1. **编译期强类型安全**：路由目标通过 Kotlin `kotlinx.serialization` 的 `@Serializable` 注解标记为强类型类或对象，入参类型不匹配在编译期即阻断，杜绝运行时崩溃。
2. **参数自动序列化解包**：无需手写从 `NavBackStackEntry.arguments` 获取 Bundle 字符串再手动转型，Navigation 框架自动完成映射。
3. **深度集成 DeepLink**：支持通过强类型路由直接配置深链唤醒匹配规则。

### 5.2 强类型路由定义规范

```kotlin
// navigation/AppRoute.kt
package com.fuxincanrui.app.navigation

import kotlinx.serialization.Serializable

sealed interface AppRoute {
    // 认证相关
    @Serializable
    data object Login : AppRoute

    @Serializable
    data object Register : AppRoute

    // 主业务相关
    @Serializable
    data object HomeGraph : AppRoute

    @Serializable
    data class DeviceDetail(val deviceId: String) : AppRoute

    @Serializable
    data class DeviceSettings(val deviceId: String) : AppRoute

    @Serializable
    data class SceneEditor(val sceneId: String? = null) : AppRoute

    // 个人中心
    @Serializable
    data object Profile : AppRoute
}
```

### 5.3 导航宿主与路由注册（`AppNavHost`）

```kotlin
// navigation/AppNavHost.kt
package com.fuxincanrui.app.navigation

import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.navigation.NavHostController
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.toRoute
import com.fuxincanrui.app.features.auth.login.LoginRoute
import com.fuxincanrui.app.features.auth.register.RegisterRoute
import com.fuxincanrui.app.features.device.detail.DeviceDetailRoute
import com.fuxincanrui.app.features.home.HomeMainRoute

@Composable
fun AppNavHost(
    navController: NavHostController,
    startDestination: AppRoute,
    modifier: Modifier = Modifier
) {
    NavHost(
        navController = navController,
        startDestination = startDestination,
        modifier = modifier
    ) {
        composable<AppRoute.Login> {
            LoginRoute(
                onNavigateToHome = {
                    navController.navigate(AppRoute.HomeGraph) {
                        popUpTo(AppRoute.Login) { inclusive = true }
                    }
                }
            )
        }

        composable<AppRoute.Register> {
            RegisterRoute(
                onNavigateBack = { navController.popBackStack() }
            )
        }

        composable<AppRoute.HomeGraph> {
            HomeMainRoute(
                onNavigateToDeviceDetail = { deviceId ->
                    navController.navigate(AppRoute.DeviceDetail(deviceId = deviceId))
                }
            )
        }

        composable<AppRoute.DeviceDetail> { backStackEntry ->
            val route = backStackEntry.toRoute<AppRoute.DeviceDetail>()
            DeviceDetailRoute(
                deviceId = route.deviceId,
                onNavigateBack = { navController.popBackStack() }
            )
        }
    }
}
```

### 5.4 路由八条铁律

1. **严禁使用旧版裸写字符串路由**：严禁调用 `navController.navigate("device_detail/$id")`，必须使用 `@Serializable` 路由实例。
2. **多 Tab 保持导航状态**：底导 Tab 切换必须配置 `popUpTo(navController.graph.findStartDestination().id) { saveState = true }` 与 `restoreState = true`，防止页面状态丢失。
3. **路由参数精简化**：强类型路由仅允许传递基本业务 ID（如 `deviceId: String`），严禁跨路由传递复杂可变大对象。
4. **单 Activity 原生架构**：全应用严格执行单 Activity（`MainActivity`）架构，所有页面均为 Composable 组件，杜绝无意义创建多个 Activity。
5. **系统返回键防护（BackHandler）**：表单编辑界面必须使用 `BackHandler(enabled = hasUnsavedChanges)` 拦截返回事件并弹出未保存提醒对话框。
6. **转场动效统一规范**：导航转场一律配置平滑的标准横向滑入滑出或淡入淡出动画，严禁无过渡突兀跳切。
7. **DeepLink 统一调度**：外部 Scheme 或 App Links 必须由统一的深链调度层校验合法性后再跳转目标路由。
8. **导航事件防双击抖动**：导航跳转回调必须在 UI 层做防抖限制，防止用户快速连击导致目标页面在回退栈中被压入多次。

---

## 六、网络层（请求对象模式）

所有 API 请求封装为继承自 `ApiRequest<T>` 接口的独立强类型请求对象。底层 OkHttp 调度器、请求拦截、统一签名与通用响应外壳解包统一收敛在 `core/network`。

### 6.1 请求基类与接口声明

```kotlin
// core/network/ApiRequest.kt
package com.fuxincanrui.app.core.network

enum class HttpMethod { GET, POST, PUT, DELETE, PATCH }

interface ApiRequest<T> {
    val path: String
    val method: HttpMethod get() = HttpMethod.POST
    val headers: Map<String, String> get() = emptyMap()
    val queryParams: Map<String, String> get() = emptyMap()
    val body: Any? get() = null
    val requiresAuth: Boolean get() = true
    val timeoutSeconds: Long get() = 15L

    fun parse(jsonString: String): T
}
```

### 6.2 业务请求对象示例

```kotlin
// features/auth/data/requests/LoginRequest.kt
package com.fuxincanrui.app.features.auth.data.requests

import com.fuxincanrui.app.core.network.ApiRequest
import com.fuxincanrui.app.core.network.HttpMethod
import com.fuxincanrui.app.features.auth.data.models.AuthToken
import kotlinx.serialization.Serializable
import kotlinx.serialization.json.Json

class LoginRequest(
    private val email: String,
    private val password: String
) : ApiRequest<AuthToken> {

    override val path: String = "/api/v1/auth/login"
    override val method: HttpMethod = HttpMethod.POST
    override val requiresAuth: Boolean = false

    @Serializable
    private data class LoginBody(val email: String, val password: String)

    override val body: Any = LoginBody(email, password)

    override fun parse(jsonString: String): AuthToken {
        return Json { ignoreUnknownKeys = true }.decodeFromString<AuthToken>(jsonString)
    }
}
```

### 6.3 统一响应体外壳解包与 ApiClient

后端标准响应外壳契约遵循：
```json
{
  "code": 200,
  "message": "success",
  "data": { ... }
}
```

`ApiClient` 负责自动剥离外壳，将内层的 `data` 字符串交付给具体 Model 的 `parse` 方法：

```kotlin
// core/network/ApiResponseEnvelope.kt
package com.fuxincanrui.app.core.network

import kotlinx.serialization.Serializable
import kotlinx.serialization.json.JsonElement

@Serializable
data class ApiResponseEnvelope(
    val code: Int,
    val message: String,
    val data: JsonElement? = null
)

// core/network/ApiClient.kt
package com.fuxincanrui.app.core.network

import com.fuxincanrui.app.core.error.AppException
import com.fuxincanrui.app.core.storage.KeychainStorage
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import kotlinx.serialization.json.Json
import okhttp3.*
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.RequestBody.Companion.toRequestBody
import java.io.IOException
import java.net.SocketTimeoutException
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class ApiClient @Inject constructor(
    private val okHttpClient: OkHttpClient,
    private val storage: KeychainStorage,
    private val baseUrl: String
) {
    private val json = Json { ignoreUnknownKeys = true }

    suspend fun <T> send(request: ApiRequest<T>): T = withContext(Dispatchers.IO) {
        val httpUrlBuilder = HttpUrl.parse(baseUrl + request.path)?.newBuilder()
            ?: throw AppException.InvalidUrl

        request.queryParams.forEach { (k, v) ->
            httpUrlBuilder.addQueryParameter(k, v)
        }

        val requestBuilder = Request.Builder()
            .url(httpUrlBuilder.build())

        request.headers.forEach { (k, v) ->
            requestBuilder.addHeader(k, v)
        }

        if (request.requiresAuth) {
            val token = storage.getAccessToken() ?: throw AppException.Unauthorized
            requestBuilder.addHeader("Authorization", "Bearer $token")
        }

        val mediaType = "application/json; charset=utf-8".toMediaType()
        val requestBody = request.body?.let {
            json.encodeToString(kotlinx.serialization.serializer(it::class.java), it).toRequestBody(mediaType)
        }

        when (request.method) {
            HttpMethod.GET -> requestBuilder.get()
            HttpMethod.POST -> requestBuilder.post(requestBody ?: "".toRequestBody(mediaType))
            HttpMethod.PUT -> requestBuilder.put(requestBody ?: "".toRequestBody(mediaType))
            HttpMethod.DELETE -> requestBuilder.delete(requestBody)
            HttpMethod.PATCH -> requestBuilder.patch(requestBody ?: "".toRequestBody(mediaType))
        }

        val response: Response
        try {
            response = okHttpClient.newCall(requestBuilder.build()).execute()
        } catch (e: SocketTimeoutException) {
            throw AppException.Timeout
        } catch (e: IOException) {
            throw AppException.NetworkError(e.localizedMessage ?: "Network error")
        }

        if (!response.isSuccessful) {
            if (response.code == 401) throw AppException.Unauthorized
            throw AppException.ServerError(response.code, "HTTP Error: ${response.code}")
        }

        val rawBody = response.body?.string() ?: throw AppException.ParseError("Empty response body")

        try {
            val envelope = json.decodeFromString<ApiResponseEnvelope>(rawBody)
            if (envelope.code != 200) {
                throw AppException.BusinessError(envelope.code, envelope.message)
            }
            val dataElement = envelope.data?.toString() ?: throw AppException.ParseError("Missing data payload")
            return@withContext request.parse(dataElement)
        } catch (e: AppException) {
            throw e
        } catch (e: Throwable) {
            throw AppException.ParseError(e.localizedMessage ?: "Failed to decode envelope")
        }
    }
}
```

### 6.4 统一异常体系（`AppException`）

```kotlin
// core/error/AppException.kt
package com.fuxincanrui.app.core.error

sealed class AppException(override val message: String) : Exception(message) {
    data object InvalidUrl : AppException("Invalid API URL")
    data class NetworkError(val detail: String) : AppException(detail)
    data object Timeout : AppException("Network timeout")
    data object Unauthorized : AppException("Unauthorized session")
    data object Forbidden : AppException("Forbidden access")
    data class ServerError(val code: Int, val detail: String) : AppException(detail)
    data class BusinessError(val code: Int, val detail: String) : AppException(detail)
    data class ParseError(val detail: String) : AppException(detail)
    data object Cancelled : AppException("Task was cancelled")
    data class Unknown(val detail: String) : AppException(detail)
}
```

---

## 七、国际化方案（按 Feature 隔离的 Android 资源）

项目采用 Android 原生资源系统并按 Feature 进行物理文件隔离。

### 7.1 核心设计与优势

1. **Feature 物理隔离**：每个 Feature 拥有独立的字符串资源文件（如 `res/values/strings_auth.xml`、`res/values/strings_home.xml`），避免单一大文件合并冲突。
2. **多语言严格对齐**：基准英文资源 `res/values/strings_*.xml` 与简体中文资源 `res/values-zh-rCN/strings_*.xml` 保持 100% 对应。
3. **参数格式化与复数支持**：原生处理各类语言复数规则（`<plurals>`）与动态参数格式化。

### 7.2 国际化七条铁律

1. **严禁硬编码任何用户可见的自然语言文本**：Composable 中显示的所有文案必须通过 `stringResource(R.string.xxx)` 读取。
2. **命名空间与前缀规范**：资源 Name 一律采用模块前缀 + 小写下划线命名法，如 `auth_login_title`、`device_offline_hint`。
3. **通用文案集中管理**："确定"、"取消"、"重试" 等全局词汇放入主 `strings.xml`，严禁在各个 Feature 中重复定义。
4. **动态多语言即时切换**：应用内切换语言时，通过更新 Application 的 `Configuration` 与本地存储即时生效。
5. **动态参数占位符严谨声明**：如 `<string name="device_count">已连接 %1$d 台设备</string>`。
6. **双语 Baseline 覆盖**：每次提交代码必须确保基准英文与简体中文双向 100% 翻译覆盖。
7. **CI 门禁检查**：CI 自动扫描是否存在未本地化的硬编码字符串。

---

## 八、UI 与交互规范

### 8.1 设计系统与 Design Tokens（`theme/`）

严格对接《FUXIN CANRUI 智能照明 App 统一设计系统规范》，在 Compose 中将其固化为设计令牌。

#### 8.1.1 间距系统（`Spacing.kt`）

基于标准的 **4dp / 8dp 黄金网格**：

```kotlin
// theme/Spacing.kt
package com.fuxincanrui.app.theme

import androidx.compose.ui.unit.dp

object AppSpacing {
    val xxs = 2.dp
    val xs = 4.dp
    val sm = 8.dp
    val md = 12.dp
    val lg = 16.dp   // 标准卡片内边距
    val xl = 20.dp
    val xxl = 24.dp
    val xxxl = 32.dp

    val screenMargin = 16.dp // 屏幕两侧安全边距
}
```

#### 8.1.2 圆角系统（`Shape.kt`）

```kotlin
// theme/Shape.kt
package com.fuxincanrui.app.theme

import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.ui.unit.dp

object AppRadius {
    val sm = RoundedCornerShape(8.dp)     // 小型标签
    val md = RoundedCornerShape(12.dp)    // 输入框、操作按键
    val card = RoundedCornerShape(16.dp)  // 标准设备卡片
    val pill = RoundedCornerShape(9999.dp)// 胶囊全圆角
}
```

#### 8.1.3 触控热区防御（48 × 48 dp 铁律）

所有可点击控件的视觉尺寸无论多小，其实际触摸响应区域必须至少满足 **48 × 48 dp**：

```kotlin
IconButton(
    onClick = onDismiss,
    modifier = Modifier.size(48.dp) // 保证触控热区
) {
    Icon(
        imageVector = Icons.Default.Close,
        contentDescription = stringResource(R.string.close),
        modifier = Modifier.size(20.dp)
    )
}
```

---

### 8.2 核心通用组件封装参考（`shared/components/`）

#### 1. 通用操作按钮 `AppButton`

```kotlin
// shared/components/AppButton.kt
package com.fuxincanrui.app.shared.components

import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import com.fuxincanrui.app.theme.AppRadius

@Composable
fun AppButton(
    text: String,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    enabled: Boolean = true,
    isLoading: Boolean = false
) {
    Button(
        onClick = onClick,
        enabled = enabled && !isLoading,
        modifier = modifier
            .fillMaxWidth()
            .height(48.dp),
        shape = AppRadius.md,
        colors = ButtonDefaults.buttonColors(
            containerColor = MaterialTheme.colorScheme.primary,
            contentColor = MaterialTheme.colorScheme.onPrimary
        )
    ) {
        if (isLoading) {
            CircularProgressIndicator(
                modifier = Modifier.size(24.dp),
                color = MaterialTheme.colorScheme.onPrimary,
                strokeWidth = 2.dp
            )
        } else {
            Text(text = text, style = MaterialTheme.typography.titleMedium)
        }
    }
}
```

#### 2. 通用四态闭环容器 `AsyncPhaseView`

```kotlin
// shared/components/AsyncPhaseView.kt
package com.fuxincanrui.app.shared.components

import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.res.stringResource
import androidx.compose.ui.unit.dp
import com.fuxincanrui.app.R
import com.fuxincanrui.app.core.network.AsyncUiState
import com.fuxincanrui.app.theme.AppSpacing

@Composable
fun <T> AsyncPhaseView(
    state: AsyncUiState<T>,
    onRetry: () -> Unit,
    modifier: Modifier = Modifier,
    content: @Composable (T) -> Unit
) {
    Box(
        modifier = modifier.fillMaxSize(),
        contentAlignment = Alignment.Center
    ) {
        when (state) {
            is AsyncUiState.Idle, is AsyncUiState.Loading -> {
                CircularProgressIndicator()
            }
            is AsyncUiState.Error -> {
                Column(
                    horizontalAlignment = Alignment.CenterHorizontally,
                    verticalArrangement = Arrangement.Center,
                    modifier = Modifier.padding(AppSpacing.lg)
                ) {
                    Text(
                        text = state.exception.message,
                        style = MaterialTheme.typography.bodyMedium,
                        color = MaterialTheme.colorScheme.onSurfaceVariant
                    )
                    Spacer(modifier = Modifier.height(AppSpacing.md))
                    Button(onClick = onRetry) {
                        Text(text = stringResource(R.string.retry))
                    }
                }
            }
            is AsyncUiState.Success -> {
                content(state.data)
            }
        }
    }
}
```

---

## 九、工程基建与质量规范

### 9.1 日志系统（`Timber`）

彻底禁止使用原生的 `Log.d()` 与 `println()`：

```kotlin
// core/logging/AppLogger.kt
package com.fuxincanrui.app.core.logging

import timber.log.Timber

object AppLogger {
    fun d(message: String, vararg args: Any?) = Timber.d(message, *args)
    fun i(message: String, vararg args: Any?) = Timber.i(message, *args)
    fun w(message: String, vararg args: Any?) = Timber.w(message, *args)
    fun e(t: Throwable?, message: String, vararg args: Any?) = Timber.e(t, message, *args)
}
```

### 9.2 自动化测试规范

```kotlin
// test/java/com/fuxincanrui/app/features/auth/LoginViewModelTest.kt
package com.fuxincanrui.app.features.auth

import app.cash.turbine.test
import com.fuxincanrui.app.features.auth.data.models.AuthToken
import com.fuxincanrui.app.features.auth.data.repository.AuthRepository
import com.fuxincanrui.app.features.auth.login.LoginUiAction
import com.fuxincanrui.app.features.auth.login.LoginViewModel
import io.mockk.coEvery
import io.mockk.mockk
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.*
import org.junit.jupiter.api.AfterEach
import org.junit.jupiter.api.Assertions.*
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test

@OptIn(ExperimentalCoroutinesApi::class)
class LoginViewModelTest {

    private val testDispatcher = StandardTestDispatcher()
    private val repository = mockk<AuthRepository>()
    private lateinit var viewModel: LoginViewModel

    @BeforeEach
    fun setup() {
        Dispatchers.setMain(testDispatcher)
        viewModel = LoginViewModel(repository)
    }

    @AfterEach
    fun tearDown() {
        Dispatchers.resetMain()
    }

    @Test
    fun `when email and password are valid, isInputValid is true`() {
        viewModel.onAction(LoginUiAction.EmailChanged("user@example.com"))
        viewModel.onAction(LoginUiAction.PasswordChanged("123456"))

        assertTrue(viewModel.uiState.value.isInputValid)
    }

    @Test
    fun `when submit succeeds, transition to success state`() = runTest(testDispatcher) {
        coEvery { repository.login(any(), any()) } returns AuthToken("mock_access_token")

        viewModel.onAction(LoginUiAction.EmailChanged("user@example.com"))
        viewModel.onAction(LoginUiAction.PasswordChanged("123456"))

        viewModel.uiState.test {
            // 初始状态
            val initial = awaitItem()
            assertFalse(initial.isSuccess)

            viewModel.onAction(LoginUiAction.Submit)

            // 加载中状态
            val loading = awaitItem()
            assertTrue(loading.isLoading)

            // 成功状态
            val success = awaitItem()
            assertTrue(success.isSuccess)
            assertFalse(success.isLoading)
        }
    }
}
```

---

## 十、新增 Feature 的标准流程

1. **创建物理目录**：在 `features/` 下新建 `<feature>/`，包含 `data/`（requests、models、repository）、`di/`。
2. **定义数据契约与仓储**：
   - 编写遵循 `ApiRequest` 接口的强类型 Request；
   - 声明不可变数据模型 `@Serializable data class`；
   - 声明 `XxxRepository` 接口并在实现类中调用 `ApiClient`。
3. **编写业务 ViewModel**：
   - 标注 `@HiltViewModel` 与 `@Inject`；
   - 声明不可变 `UiState` 数据类；
   - 通过 `viewModelScope.launch` 调度协程更新 `_uiState`。
4. **编写 Compose 页面（Colocation）**：
   - 建立页面目录（如 `features/<feature>/login/`）；
   - 拆分有状态 Route 与无状态 Screen；
   - 保证 48dp 触控热区、软键盘避让与四态闭环。
5. **注册强类型路由**：
   - 在 `navigation/AppRoute.kt` 增加 `@Serializable` 路由；
   - 在 `AppNavHost` 注册 `composable<AppRoute.Xxx>`。
6. **补齐多语言与测试**：
   - 在 `res/values/strings_<feature>.xml` 与中文目录下录入词条；
   - 编写 ViewModel 状态流单元测试。

---

## 十一、检查清单（Code Review 用）

- [ ] **架构合规**：View 中是否纯粹声明式渲染，绝无直接请求网络、修改 Repository 或编写业务逻辑的行为？
- [ ] **协程安全**：耗时任务是否放在 `Dispatchers.IO` 中？ViewModel 中是否由 `viewModelScope` 发起？
- [ ] **不可变性**：数据模型与 UI 状态类是否为不可变 `data class`？
- [ ] **触控与无障碍**：所有图标点击热区是否均达到 48 × 48 dp？
- [ ] **文本溢出防御**：动态文本是否包裹了 `maxLines` 与 `TextOverflow.Ellipsis`？
- [ ] **设计系统对齐**：是否 100% 杜绝了硬编码魔法像素（必须采用 `AppSpacing`）与硬编码色值？
- [ ] **国际化覆盖**：界面文案是否全部放入对应 Feature 的 `strings.xml`，杜绝硬编码字符串？
- [ ] **日志合规**：生产代码中是否已彻底杜绝 `println` 与 `Log.d`，统一采用 `Timber`？
- [ ] **质量门禁**：`ktlint` 与 `detekt` 0 警告 / 0 错误通过，单元测试全绿。

---

## 十二、AI 开发协作规范（AI_RULES 体系与落地）

### 12.1 核心架构定位

在 Android 原生工程中，所有 AI 助手（Claude、Cursor、Copilot、Gemini、Antigravity）必须严格以工程根目录下的 `AI_RULES/RULES.md` 为统一总控清单。

### 12.2 十条绝对红线（Hard Bans）

1. **严禁私自引入未经批准的 Gradle 第三方依赖**。
2. **严禁在 Compose 中手写旧版基于字符串拼接的导航路由**。
3. **严禁在 ViewModel 中持有 `Context`、`View` 或任何 Compose 视图实例**。
4. **严禁在 Composable 渲染流中直接触发产生副作用的网络调用**。
5. **严禁在 ViewModel 中暴露可变的 `MutableStateFlow` 给 View**。
6. **严禁硬编码文本、硬编码颜色值与硬编码固定间距数值**。
7. **严禁在生产代码中使用 `println` 或原生 `Log.d`**。
8. **严禁在 ViewModel 中省略 Hilt `@Inject` 构造器依赖注入**。
9. **严禁破坏 Feature 隔离直接跨模块引用私有类或视图**。
10. **严禁交付未通过 `ktlint` 静态检查或编译报错的代码**。
