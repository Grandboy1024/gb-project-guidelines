## 一、技术选型

| 类别 | 选择 | 说明 |
| --- | --- | --- |
| 最低支持版本 | **iOS 16.0+** | 全工程统一以 **iOS 16.0** 为最低部署目标（Deployment Target），严禁直接裸用 iOS 17+ 专属 API（如 `@Observable`、`SwiftData`、`ContentUnavailableView`） |
| 语言与编译器 | **Swift 5.9+ / Swift 6** | 开启严格并发检查（Strict Concurrency Checking: `Complete`），全面采用现代并发机制（`async`/`await`、`actor`、`Task`、`Sendable`） |
| 架构模式 | **Clean MVVM** | 见第三节，View (SwiftUI) / ViewModel (`ObservableObject` + `@MainActor`) / Repository / ApiClient / Storage |
| UI 框架 | **SwiftUI** | iOS 16.0+，全声明式 UI，组件原子化解耦，支持 Dynamic Island 与安全区自适应 |
| 状态管理 | **`ObservableObject` + `@Published`** | 见第四节，基于 `@StateObject` / `@ObservedObject` / `@EnvironmentObject` 构建单向响应式数据流，配合细粒度子视图拆分避免过度重绘 |
| 依赖注入 | **Protocol-based Dependency Container / Environment** | 基于 Swift 协议抽象的轻量容器化注入（通过 `@EnvironmentObject` 或显式构造传参），严禁强耦合第三方庞大框架，测试期支持 Mock 无缝替换 |
| 路由导航 | **`NavigationStack` + `NavigationPath`** | 见第五节，iOS 16+ 原生强类型路由 Enum（遵循 `Hashable` / `Codable` / `Identifiable`），统一支持 DeepLink 深度唤醒与栈式导航管控 |
| 网络层 | **`URLSession` + 请求对象模式 (`ApiRequest`)** | 见第六节，原生 `async/await` 调度，内置统一响应体外壳解包、拦截器管道、`actor` 并发安全的 Token 自动续期与 `Task` 取消 |
| 数据类 / 序列化 | **Swift `Codable`** (`Decodable` / `Encodable`) | 模型全部基于不可变值类型 `struct`，强约束遵循 `Sendable` |
| 本地存储 | **`Keychain`**（安全凭证/Token）+ **`UserDefaults`**（轻量配置）+ **`CoreData`**（结构化缓存） | 严禁明文持久化 Token 与敏感机密；结构化数据库统一基于 iOS 16 兼容的 `CoreData` |
| 错误处理 | **自定义 `AppException` 密封枚举 + `AsyncPhase<T>`** | 见第六节与第四节，强类型穷尽错误捕获，携带 `traceId` 与 `isRetryable`，不向上层暴露底层 `URLError` |
| 国际化 | **Apple String Catalogs (`.xcstrings`)**，**按 feature 隔离** | 见第七节，编译期打包向后兼容 iOS 16+，按业务 Feature 独立配置维护 |
| UI 规范 / 主题 / 间距 | **HIG (Human Interface Guidelines)** + `AppTheme` (Design Tokens) | 见第八节，定义 `AppSpacing`、`AppRadius`、`AppColors` 语义色、Dynamic Type 语义字阶，规范四态闭环与边界防御 |
| 环境配置 | **`xcconfig` + 集中环境配置 (`Core/Config/Environment.swift`)** | 见第九节，支持 Debug / Staging / Release 动态配置切换 |
| 日志系统 | **Apple Unified Logging (`os.Logger`)** | 统一 Subsystem 与 Category，支持隐私脱敏（`privacy: .private`），严禁 `print()` |
| 测试框架 | **`Swift Testing`** (`@Test`, `#expect`) + **`XCTest`** | 见第九节，覆盖 Repository、ViewModel 与 View 交互测试（向下兼容 iOS 16 运行时） |
| 代码规范 / Lint | **`SwiftLint`** + **`swift-format`** | 见第九节 9.4，开启最严格安全规则与并发检查，CI 强校验阻断 |
| 依赖包管理 | **Swift Package Manager (`SPM`)** | 原生集成于 Xcode，优先零外部依赖；禁止新工程引入 CocoaPods；无 SPM 支持的闭源 SDK 仅允许以本地 `XCFramework` 评审接入 |

---

## 二、目录结构

```
my-ios-app/
├── AI_RULES/                     # AI 协作主规范体系（单一真实来源，所有 AI 工具共用）
│   ├── RULES.md                  # 规范总入口 / 规范清单 (Manifest)
│   ├── core/                     # 核心规范：MASTER(行为), WORKFLOW(流程), COMMUNICATION(沟通)
│   ├── development/              # 开发规范：CODING(编码), ARCHITECTURE(架构), TESTING(测试)
│   ├── project/                  # 项目知识：CONTEXT(背景), DECISIONS(技术决策), STATUS(进度)
│   └── CHANGELOG.md              # 规范变更记录
├── AGENTS.md                     # Codex / Copilot / OpenDevin 平台适配入口
├── CLAUDE.md                     # Claude Code 平台适配入口
├── GEMINI.md                     # Gemini CLI / Antigravity 平台适配入口
├── App/
│   ├── MyApp.swift               # 应用入口：@main 结构体，配置 RootView、AppSession 与依赖注入
│   ├── AppBootstrap.swift        # 初始化装配（日志通道、本地持久化、网络适配、监控拦截）
│   ├── Navigation/
│   │   ├── AppRouter.swift       # 全局路由栈状态持有者（ObservableObject + NavigationPath）
│   │   ├── AppRoute.swift        # 全局强类型路由枚举定义（Hashable / Codable / Identifiable）
│   │   ├── AppRouteViewFactory.swift # 路由目标分发器（navigationDestination / sheet / fullScreenCover）
│   │   └── DeepLinkHandler.swift # Universal Links 与 Scheme 解析调度
│   └── Theme/                    # 主题、色彩系统、字体与设计令牌
│       ├── AppColors.swift       # 语义色资产定义（支持 Light/Dark 模式自适应）
│       ├── AppSpacing.swift      # 4pt/8pt 网格间距常量系统
│       ├── AppRadius.swift       # 统一圆角常量系统
│       └── AppTypography.swift   # 统一排版与 Dynamic Type 语义字阶（iOS 16+）
├── Core/                         # 与业务完全解耦的基础设施
│   ├── Config/
│   │   └── Environment.swift     # 环境配置加载器（基于 xcconfig 映射）
│   ├── Network/
│   │   ├── ApiClient.swift       # URLSession 调度中枢，统一发请求的唯一出口
│   │   ├── ApiRequest.swift      # 请求基类协议（声明 Path、Method、Body 编码、解析泛型）
│   │   ├── ApiResponseEnvelope.swift # 统一响应体外壳与 EmptyResponse 定义
│   │   ├── AuthTokenCoordinator.swift # 基于 actor 的并发安全 Token 刷新协调器
│   │   ├── Interceptors/         # 统一签名头、鉴权 Token 附加、请求日志与重试拦截器
│   │   └── ApiEndpoint.swift     # 域名与 Host 动态配置
│   ├── BLE/                      # CoreBluetooth 封装（actor + AsyncThrowingStream）
│   ├── Storage/
│   │   ├── KeychainStorage.swift # Keychain 安全存储封装（存储 Token / 加密密钥）
│   │   ├── AppDefaults.swift     # UserDefaults 类型安全属性包装器
│   │   └── Database/             # CoreData 上下文与后台持久化容器封装
│   ├── Error/
│   │   └── AppException.swift    # 全局密封错误枚举（网络、解析、权限、业务错误）
│   ├── Logging/
│   │   └── AppLogger.swift       # 基于 os.Logger 的分类日志中枢
│   ├── Utils/
│   │   └── HapticManager.swift   # 统一触觉反馈封装
│   └── DI/
│       └── DependencyContainer.swift # 依赖注入中枢容器（通过 @EnvironmentObject 注入）
├── Shared/                       # 跨 Feature 高度复用的通用模块
│   ├── Components/               # 通用基础 UI 组件（AppButton、AppTextField、AsyncPhaseView、AppEmptyView）
│   ├── Extensions/               # Foundation / SwiftUI 扩展（View+DismissKeyboard 等）
│   ├── Modifiers/                # 常用通用 ViewModifier（防抖、加载弹层、安全区边界）
│   └── Resources/
│       ├── Common.xcstrings      # 全局通用文案（确定、取消、重试、完成）
│       └── Errors.xcstrings      # 全局统一错误兜底文案
├── Features/                     # 业务模块目录（完全高内聚、物理隔离）
│   └── Auth/                     # 认证模块 Feature
│       ├── Data/                 # 数据层（本 Feature 内部共享）
│       │   ├── Requests/         # LoginRequest 等强类型请求对象
│       │   ├── Models/           # Sendable & Codable 业务数据结构体（User、AuthToken）
│       │   └── AuthRepository.swift # 业务仓储实现及协议定义
│       ├── Resources/            # Feature 专用本地化与静态资源
│       │   └── Auth.xcstrings    # 本 Feature 专用的 String Catalog 国际化文案
│       ├── AuthSession.swift     # 跨页面常驻的会话级状态（登录态、当前用户，由 App 注入）
│       ├── Login/                # 登录页面就近聚合
│       │   ├── LoginView.swift   # SwiftUI 视图组件
│       │   ├── LoginViewModel.swift # 基于 ObservableObject 与 @MainActor 的视图模型
│       │   └── Components/       # 登录页私有子组件（如第三方登录按键区）
│       └── Register/             # 注册页面
│           ├── RegisterView.swift
│           └── RegisterViewModel.swift
├── Configurations/               # xcconfig 环境配置文件（Debug / Release）
├── PrivacyInfo.xcprivacy         # Apple 隐私清单（Required Reason APIs 声明）
└── Tests/                        # 自动化测试工程（镜像对应 Features / Core）
    ├── Features/
    │   └── Auth/
    │       ├── LoginViewModelTests.swift
    │       └── AuthRepositoryTests.swift
    └── Core/
        └── Network/
            └── ApiClientTests.swift
```

### 目录组织规则

1. **就近聚合（Colocation），消除深层冗余**：摒弃传统庞大的 `Controller/`、`Presenter/` 跨越式分包。页面直接在 Feature 下按功能建目录（如 `Login/`、`Register/`）。一个页面的 View（`LoginView.swift`）、ViewModel（`LoginViewModel.swift`）和私有组件（`Components/`）聚合在一起，开发维护无需在工程树中频繁上下翻找。
2. **异步状态标准化管理**：绝大多数页面的异步操作统一采用 `AsyncPhase<T>`（封装 Idle / Loading / Success / Failure），页面操作通过 Swift 现代化 `Task` 结构化并发调度，严格配合 `@MainActor` 隔离主线程，杜绝状态不一致。
3. **数据层保持在 Feature 级**：`Data/`（Repository、Requests、Models）由整个 Feature 共享，不按单一页面下沉。模型采用遵循 `Codable` 与 `Sendable` 的原生 `struct`，集中在 `Data/` 能保证边界清晰并杜绝不同 Feature 间互相破坏性耦合。
4. **生命周期与资源自释放**：
   - 页面级 ViewModel 直接由页面通过 `@StateObject` 声明并依托 SwiftUI 视图生命周期自动释放；
   - 跨页面常驻的会话级状态（如 `AppSession`、`AppRouter`），定义在 App 核心容器并通过 `@EnvironmentObject` 注入，保证唯一真实来源。
5. **组件归宿清晰明确**：
   - 仅单个页面私有的组件放该页面目录下的 `Components/`；
   - 本 Feature 内两个以上页面共用的组件放 `Features/<feature>/Components/`；
   - 跨两个以上 Feature 共用的通用原子组件必须提升到根目录 `Shared/Components/`。
6. **国际化完全自闭环**：采用 Apple 原生 **String Catalogs (`.xcstrings`)**，按 Feature 物理隔离存放于 `Features/<Feature>/Resources/`，全局通用与错误文案存放于 `Shared/Resources/`。借助编译期校验与强类型封装，严禁跨 Feature 乱串文案。
7. **强类型栈式路由**：基于 iOS 16+ 原生 `NavigationStack` + `NavigationPath`，所有路由目标统一在 `AppRoute` 枚举中以强类型枚举值声明（携带强类型参数并遵循 `Hashable`、`Identifiable`），严禁手写动态拼接字符串进行页面跳转。
8. **设计系统与硬编码防御**：组件间距一律使用 `AppSpacing` 常量；字体、圆角与颜色必须通过 `AppTheme` 与语义色获取，严禁在 View 中硬编码诸如 `Color(hex: 0x123456)` 或固定数值内边距。
9. **测试目录与源码镜像**：测试文件命名以 `Tests.swift` 结尾，目录结构与被测源码保持 1:1 镜像对齐。
10. **AI 规范单一真实来源**：根目录下的 `AI_RULES/` 为唯一真理源，`AGENTS.md`、`CLAUDE.md`、`GEMINI.md` 仅作为各平台的适配入口。

---

## 三、MVVM 落地规则

### 3.1 角色对应

| 角色 | 典型实现 | 职责 |
| --- | --- | --- |
| **View** | SwiftUI `struct: View` | 声明式渲染 UI、转发用户手势、响应状态驱动页面转场/弹窗，绝不含复杂业务逻辑与网络代码 |
| **ViewModel** | `@MainActor final class XxxViewModel: ObservableObject` | 持有不可变 UI 状态（`@Published private(set)`），调度异步 `Task`，调用 Repository，向 View 暴露响应式数据 |
| **Model** | Repository + `struct Model: Codable, Sendable` | Repository 是 ViewModel 获取数据的唯一网关，屏蔽底层网络、缓存与存储细节 |

**依赖方向（单向流动）**：

```
View (SwiftUI) → ViewModel (ObservableObject) → Repository → ApiClient / Storage
```

禁止反向引用；View 严禁直接绕过 ViewModel 调用 Repository 或 ApiClient。

---

### 3.2 示例：登录

#### 1. ViewModel（基于 `ObservableObject` 与 `@MainActor`）

```swift
// Features/Auth/Login/LoginViewModel.swift
import Combine
import Foundation

@MainActor
final class LoginViewModel: ObservableObject {
    // 视图直接消费的只读状态（严格延迟本地化：仅透传 AppException）
    @Published private(set) var phase: AsyncPhase<AuthToken> = .idle
    @Published var email: String = ""
    @Published var password: String = ""

    private let authRepository: AuthRepositoryProtocol
    private var currentTask: Task<Void, Never>?

    init(authRepository: AuthRepositoryProtocol) {
        self.authRepository = authRepository
    }

    var isInputValid: Bool {
        !email.trimmingCharacters(in: .whitespacesAndNewlines).isEmpty &&
        password.count >= 6
    }

    /// 供 SwiftUI 按钮触发的同步入口（内置防重入提交保护）
    func login() {
        guard isInputValid, !phase.isLoading else { return }

        currentTask = Task {
            await performLogin()
        }
    }

    /// 供 .task 修饰符或单元测试直接 await 的确定性异步入口
    func performLogin() async {
        guard isInputValid, !phase.isLoading else { return }

        phase = .loading
        do {
            let token = try await authRepository.login(email: email, password: password)
            guard !Task.isCancelled else { return }
            phase = .success(token)
        } catch let error as AppException {
            guard !Task.isCancelled else { return }
            phase = .failure(error)
        } catch {
            guard !Task.isCancelled else { return }
            phase = .failure(.unknown(error.localizedDescription))
        }
    }

    func cancelOngoingRequest() {
        currentTask?.cancel()
        currentTask = nil
        if phase.isLoading {
            phase = .idle
        }
    }
}
```

> **Swift 6 并发提示**：`deinit` 为 `nonisolated`，不可直接读取 `@MainActor` 隔离的 `currentTask`。统一由视图 `.onDisappear { viewModel.cancelOngoingRequest() }` 或 SwiftUI `.task` 生命周期驱动取消。

#### 2. Repository

```swift
// Features/Auth/Data/AuthRepository.swift
import Foundation

protocol AuthRepositoryProtocol: Sendable {
    func login(email: String, password: String) async throws -> AuthToken
}

final class AuthRepository: AuthRepositoryProtocol {
    private let apiClient: ApiClientProtocol
    private let storage: KeychainStorageProtocol

    init(apiClient: ApiClientProtocol, storage: KeychainStorageProtocol) {
        self.apiClient = apiClient
        self.storage = storage
    }

    func login(email: String, password: String) async throws -> AuthToken {
        let request = LoginRequest(email: email, password: password)
        let token = try await apiClient.send(request)
        // 缓存安全凭证至 Keychain
        try await storage.save(token.accessToken, for: .accessToken)
        return token
    }
}
```

#### 3. View (SwiftUI，页面级 View 不自带 NavigationStack)

```swift
// Features/Auth/Login/LoginView.swift
import SwiftUI

struct LoginView: View {
    @StateObject private var viewModel: LoginViewModel
    @EnvironmentObject private var appSession: AppSession
    @FocusState private var focusedField: Field?

    private enum Field: Hashable {
        case email, password
    }

    init(viewModel: @autoclosure @escaping () -> LoginViewModel) {
        _viewModel = StateObject(wrappedValue: viewModel())
    }

    var body: some View {
        ScrollView {
            VStack(spacing: AppSpacing.lg) {
                VStack(alignment: .leading, spacing: AppSpacing.xs) {
                    Text(authKey: "login_title")
                        .font(AppTypography.pageTitle)
                        .foregroundStyle(AppColors.textPrimary)

                    Text(authKey: "login_subtitle")
                        .font(AppTypography.bodyMedium)
                        .foregroundStyle(AppColors.textSecondary)
                }
                .frame(maxWidth: .infinity, alignment: .leading)

                VStack(spacing: AppSpacing.md) {
                    // 邮箱输入框
                    AppTextField(
                        text: $viewModel.email,
                        placeholder: String(localized: "email_placeholder", table: "Auth"),
                        systemImage: "envelope"
                    )
                    .textContentType(.emailAddress)
                    .keyboardType(.emailAddress)
                    .textInputAutocapitalization(.never)
                    .autocorrectionDisabled()
                    .focused($focusedField, equals: .email)
                    .submitLabel(.next)
                    .onSubmit { focusedField = .password }

                    // 密码输入框（支持明密文切换与一键清空）
                    AppTextField(
                        text: $viewModel.password,
                        placeholder: String(localized: "password_placeholder", table: "Auth"),
                        systemImage: "lock",
                        isSecure: true
                    )
                    .textContentType(.password)
                    .focused($focusedField, equals: .password)
                    .submitLabel(.done)
                    .onSubmit {
                        if viewModel.isInputValid {
                            focusedField = nil
                            viewModel.login()
                        }
                    }
                }

                // 延迟本地化：仅在 View 层渲染 AppException 文案
                if case .failure(let error) = viewModel.phase {
                    Text(error.localizedDescription)
                        .font(AppTypography.caption)
                        .foregroundStyle(AppColors.error)
                        .frame(maxWidth: .infinity, alignment: .leading)
                        .transition(.opacity.combined(with: .move(edge: .top)))
                }

                // 登录主按钮
                AppButton(
                    title: String(localized: "login_button", table: "Auth"),
                    isLoading: viewModel.phase.isLoading,
                    disabled: !viewModel.isInputValid
                ) {
                    focusedField = nil
                    viewModel.login()
                }
            }
            .padding(.horizontal, AppSpacing.screenMargin)
            .padding(.vertical, AppSpacing.xl)
        }
        .scrollDismissesKeyboard(.immediately)
        .dismissKeyboardOnTap()
        .background(AppColors.background)
        .navigationBarTitleDisplayMode(.inline)
        .toolbar {
            ToolbarItem(placement: .keyboard) {
                Button(String(localized: "done", table: "Common")) {
                    focusedField = nil
                }
            }
        }
        // iOS 16 兼容的单参数 onChange 写法
        .onChange(of: viewModel.phase) { phase in
            if case .success(let token) = phase {
                appSession.updateSession(token: token)
            }
        }
        .onDisappear {
            viewModel.cancelOngoingRequest()
        }
    }
}
```

---

### 3.3 十条硬性规则

1. **统一采用 `ObservableObject` + `@Published`（iOS 16+ 基线）**：全工程 ViewModel 统一继承 `ObservableObject`（配合 `@StateObject` 持有、`@ObservedObject` 接收、`@EnvironmentObject` 分发）；严禁在最低版本为 iOS 16.0 的工程中混用 iOS 17+ 专属 `@Observable` / `@Bindable` 宏导致编译失败。
2. **ViewModel 必须标注 `@MainActor`**：所有直接参与 UI 状态驱动的 ViewModel 类必须标注 `@MainActor`，从编译器级别切断跨线程更新 UI 导致的崩溃隐患。
3. **数据模型不可变性（Immutability）**：Model 层一律使用遵循 `Sendable` 的不可变 `struct`，禁止在 Model 内部挂载可变业务逻辑。
4. **禁止在 ViewModel 中引入 UIKit / 视图层组件**：ViewModel 内部严禁 `import SwiftUI`（仅允许引入 `Foundation`、`Combine` 等纯逻辑库），严禁持有 `UIView`、`UIViewController` 或具体的 SwiftUI 视图实例。
5. **错误多语言延迟翻译**：ViewModel 和 Repository 仅透传强类型 `AppException` 密封枚举体系，多语言本地化文案必须由 View 层在渲染时根据当前上下文和 String Catalogs 匹配呈现。
6. **Repository 屏障防御**：Repository 必须拦截底层 `URLError` 或 `DecodingError`，并转换为上层友好的 `AppException`，严禁让未处理的网络层原始异常穿透到 UI。
7. **`Domain/` 目录按需引入**：默认不设立多余的 Domain/UseCase 胶水层，Repository 直接与 ViewModel 对接；仅当某一段业务逻辑在 3 个以上 ViewModel 间高频复用时才提取独立 UseCase。
8. **Feature 物理隔离**：Feature 之间严禁交叉 `import` 或跨模块直接调用彼此私有组件；全局共享逻辑必须下沉到 `Core/` 或 `Shared/`。
9. **结构化并发取消管控**：页面退出或重入时，ViewModel 必须显式处理当前挂起的 `Task` 取消，防止后台请求回包后发生无意义的状态篡改。
10. **统一命名约定**：严格遵照 `XxxView.swift`、`XxxViewModel.swift`、`XxxRepository.swift`、`XxxRequest.swift` 规范命名，类名与文件名保持严格一致。

---

## 四、状态管理规范（SwiftUI ObservableObject 实践）

项目在 **iOS 16.0+** 基线下统一基于 **`ObservableObject` + `@Published`** 与 Swift 结构化并发（Structured Concurrency）构建单向响应式状态层。

### 4.1 核心原则与架构定位

1. **唯一真实来源（Single Source of Truth）**：
   - 视图状态严格收敛在 ViewModel 内部，UI 仅为状态的纯函数映射；用户任何交互意图均通过调用 ViewModel 暴露的公开方法改变状态。
2. **所有权与注入边界清晰**：
   - **创建与持有用 `@StateObject`**：页面首次实例化 ViewModel 时，必须使用 `@StateObject`（配合 `@autoclosure` 构造注入），确保 SwiftUI 视图刷新时不会重复销毁重建实例；
   - **子组件借用用 `@ObservedObject`**：仅当子视图需要直接观察父视图传入的外部对象时使用 `@ObservedObject`；优先推荐向纯展示子组件传递不可变值类型（`struct`）与回调闭包，避免因单一 `@Published` 变化引起大范围无意义重绘；
   - **全局共享用 `@EnvironmentObject`**：跨页面会话（`AppSession`）与路由器（`AppRouter`）统一通过 `.environmentObject(...)` 向下注入。
3. **严格主线程隔离与并发安全**：
   - 所有驱动 UI 刷新的 ViewModel 必须声明为 `@MainActor`，耗时数据计算、网络序列化与 I/O 读取全部在后台 `actor` 或异步函数中完成，避免阻塞主线程 RunLoop。

### 4.2 ViewModel 选型与声明范式

#### 4.2.1 页面级状态控制器

```swift
@MainActor
final class DeviceListViewModel: ObservableObject {
    @Published private(set) var phase: AsyncPhase<[DeviceItem]> = .idle

    private let repository: DeviceRepositoryProtocol

    init(repository: DeviceRepositoryProtocol) {
        self.repository = repository
    }

    func fetchDevices() async {
        phase = .loading
        do {
            let items = try await repository.getDevices()
            guard !Task.isCancelled else { return }
            phase = .success(items)
        } catch let err as AppException {
            guard !Task.isCancelled else { return }
            phase = .failure(err)
        } catch {
            guard !Task.isCancelled else { return }
            phase = .failure(.unknown(error.localizedDescription))
        }
    }
}
```

#### 4.2.2 全局共享会话状态（App Session）与根视图状态机

针对登录态、全局主题模式、当前用户信息等跨页面持久常驻的状态，在 `App` 根节点创建 `@StateObject` 并通过 `.environmentObject` 向下分发：

```swift
@MainActor
final class AppSession: ObservableObject {
    @Published private(set) var currentUser: UserProfile?
    @Published private(set) var isAuthenticated: Bool = false

    func updateSession(token: AuthToken, user: UserProfile? = nil) {
        if let user {
            self.currentUser = user
        }
        self.isAuthenticated = true
    }

    func logout() {
        self.currentUser = nil
        self.isAuthenticated = false
    }
}

// App/MyApp.swift 根节点切换范式
struct RootView: View {
    @EnvironmentObject private var appSession: AppSession
    @EnvironmentObject private var container: DependencyContainer

    var body: some View {
        Group {
            if appSession.isAuthenticated {
                MainTabView()
            } else {
                NavigationStack {
                    LoginView(viewModel: container.makeLoginViewModel())
                        .withAppRouter()
                }
            }
        }
        .animation(.easeInOut(duration: 0.25), value: appSession.isAuthenticated)
    }
}
```

### 4.3 异步状态高级模式（`AsyncPhase<T>` 范式）

针对所有带网络请求的界面，统一采用支持 `Sendable` 与 `Equatable` 的四态枚举封装：

```swift
// Core/Network/AsyncPhase.swift
public enum AsyncPhase<Value: Sendable>: Sendable {
    case idle
    case loading
    case success(Value)
    case failure(AppException)

    public var value: Value? {
        if case .success(let val) = self { return val }
        return nil
    }

    public var isLoading: Bool {
        if case .loading = self { return true }
        return false
    }

    public var isSuccess: Bool {
        if case .success = self { return true }
        return false
    }
}

extension AsyncPhase: Equatable where Value: Equatable {}
```

### 4.4 状态管理八条禁忌（Anti-Patterns）

1. **严禁在 View 的 `body` 中直接执行异步任务**：必须使用 `.task { ... }` 修饰符或用户事件触发，利用 SwiftUI 自动绑定的生命周期在视图离开时自动取消任务。
2. **严禁使用 `@ObservedObject` 创建 ViewModel 实例**：拥有该实例生命周期的父/当前 View 必须使用 `@StateObject`，防止视图刷新时实例被重置导致状态丢失。
3. **严禁使用全局单例任意篡改业务状态**：除基础设施（网络调度器、Keychain、Haptics）外，业务 ViewModel 与 DI 容器严禁作为无约束全局静态单例散落调用，必须由根容器或初始化流程显式注入。
4. **严禁手写通知监听广播（`NotificationCenter`）作为核心业务状态流**：视图与控制器之间的状态传递必须走 `ObservableObject` / `Combine` 响应式数据流。
5. **严禁非 `@MainActor` 线程修改 `@Published` 属性**：后台线程直接修改 `@Published` 会触发 Xcode 紫色主线程警告甚至引发随机 UI 撕裂崩溃。
6. **严禁忽视 `Task.isCancelled` 检查**：长耗时网络请求或密集循环计算后，写入状态前必须先检查 `Task.isCancelled`，避免向已销毁的页面写入陈旧脏数据。
7. **严禁混用 iOS 17+ 专属状态与存储 API**：在最低版本为 iOS 16.0 的工程中，严禁使用 `@Observable`、`@Bindable`、双参数 `.onChange(of:) { old, new in }` 或 `SwiftData`。
8. **严禁直接在 View 中嵌套复杂计算属性**：复杂的数据加工与过滤必须在 ViewModel 中提前计算好，避免在每帧渲染时重复执行高开销遍历。

---

## 五、路由规范（NavigationStack + NavigationPath 强类型路由）

项目全面摒弃已废弃的 `NavigationView` 与字符串动态路由，全量采用 iOS 16+ 官方标准的 **`NavigationStack` 配合 `NavigationPath`** 强类型栈式路由架构。

### 5.1 核心设计与优势

1. **编译期强类型校验**：路由定义为遵循 `Hashable`、`Codable` 与 `Identifiable` 的枚举，页面传参具备静态编译保障，彻底消灭字符串拼错与参数缺失崩溃。
2. **DeepLink 与状态序列化天然支持**：`NavigationPath` 原生支持序列化为 JSON，支持将整个应用导航栈持久化存储与精准深链还原。
3. **解耦页面跳转细节**：View 仅需向全局 `AppRouter` 发出"推入/弹出某个路由对象"指令，具体目标视图的匹配与转场在统一注册表中收敛。

### 5.2 强类型路由枚举定义

```swift
// App/Navigation/AppRoute.swift
import Foundation

public enum AppRoute: Hashable, Codable, Identifiable, Sendable {
    // 认证相关
    case login
    case register
    case forgotPassword

    // 主业务相关
    case deviceDetail(deviceId: String)
    case deviceSettings(deviceId: String)
    case sceneEditor(sceneId: String?)

    // 个人中心
    case userProfile
    case appSettings

    public var id: String {
        switch self {
        case .login: return "login"
        case .register: return "register"
        case .forgotPassword: return "forgotPassword"
        case .deviceDetail(let deviceId): return "deviceDetail:\(deviceId)"
        case .deviceSettings(let deviceId): return "deviceSettings:\(deviceId)"
        case .sceneEditor(let sceneId): return "sceneEditor:\(sceneId ?? "new")"
        case .userProfile: return "userProfile"
        case .appSettings: return "appSettings"
        }
    }
}
```

### 5.3 全局路由器实现（`AppRouter`）

```swift
// App/Navigation/AppRouter.swift
import SwiftUI

@MainActor
public final class AppRouter: ObservableObject {
    @Published public var path = NavigationPath()
    @Published public var presentedSheet: AppRoute?
    @Published public var presentedFullScreenCover: AppRoute?

    public init() {}

    public func push(_ route: AppRoute) {
        path.append(route)
    }

    public func pop() {
        guard !path.isEmpty else { return }
        path.removeLast()
    }

    public func popToRoot() {
        path = NavigationPath()
    }

    public func presentSheet(_ route: AppRoute) {
        presentedSheet = route
    }

    public func dismissSheet() {
        presentedSheet = nil
    }

    public func presentFullScreen(_ route: AppRoute) {
        presentedFullScreenCover = route
    }

    public func dismissFullScreen() {
        presentedFullScreenCover = nil
    }
}
```

### 5.4 路由目标分发器（`AppRouteViewFactory`）

> 严禁在 `switch route` 中添加 `default:` 分支，确保每次在 `AppRoute` 新增页面 Case 时，编译器强制检查所有路由出口已正确注册；同时必须统一接管 `.sheet(item:)` 与 `.fullScreenCover(item:)`。

```swift
// App/Navigation/AppRouteViewFactory.swift
import SwiftUI

struct AppRouteDestinationModifier: ViewModifier {
    @EnvironmentObject private var router: AppRouter
    @EnvironmentObject private var container: DependencyContainer

    @ViewBuilder
    private func destinationView(for route: AppRoute) -> some View {
        switch route {
        case .login:
            LoginView(viewModel: container.makeLoginViewModel())
        case .register:
            RegisterView(viewModel: container.makeRegisterViewModel())
        case .forgotPassword:
            ForgotPasswordView(viewModel: container.makeForgotPasswordViewModel())
        case .deviceDetail(let deviceId):
            DeviceDetailView(deviceId: deviceId)
        case .deviceSettings(let deviceId):
            DeviceSettingsView(deviceId: deviceId)
        case .sceneEditor(let sceneId):
            SceneEditorView(sceneId: sceneId)
        case .userProfile:
            UserProfileView()
        case .appSettings:
            AppSettingsView()
        }
    }

    func body(content: Content) -> some View {
        content
            .navigationDestination(for: AppRoute.self) { route in
                destinationView(for: route)
            }
            .sheet(item: $router.presentedSheet) { route in
                NavigationStack {
                    destinationView(for: route)
                }
                .environmentObject(router)
                .environmentObject(container)
            }
            .fullScreenCover(item: $router.presentedFullScreenCover) { route in
                NavigationStack {
                    destinationView(for: route)
                }
                .environmentObject(router)
                .environmentObject(container)
            }
    }
}

extension View {
    public func withAppRouter() -> some View {
        modifier(AppRouteDestinationModifier())
    }
}
```

### 5.5 多 Tab 状态保持结构（`MainTabView`）

```swift
// Features/Home/MainTabView.swift
import SwiftUI

enum AppTab: Int, Hashable, CaseIterable {
    case home = 0
    case smart = 1
    case profile = 2
}

struct MainTabView: View {
    @State private var selectedTab: AppTab = .home
    @StateObject private var homeRouter = AppRouter()
    @StateObject private var smartRouter = AppRouter()
    @StateObject private var profileRouter = AppRouter()

    var body: some View {
        TabView(selection: $selectedTab) {
            NavigationStack(path: $homeRouter.path) {
                HomeDashboardView()
                    .withAppRouter()
            }
            .environmentObject(homeRouter)
            .tabItem {
                Label(String(localized: "tab_home", table: "Main"), systemImage: "lightbulb")
            }
            .tag(AppTab.home)

            NavigationStack(path: $smartRouter.path) {
                SmartSceneListView()
                    .withAppRouter()
            }
            .environmentObject(smartRouter)
            .tabItem {
                Label(String(localized: "tab_smart", table: "Main"), systemImage: "wand.and.stars")
            }
            .tag(AppTab.smart)

            NavigationStack(path: $profileRouter.path) {
                ProfileView()
                    .withAppRouter()
            }
            .environmentObject(profileRouter)
            .tabItem {
                Label(String(localized: "tab_profile", table: "Main"), systemImage: "person.crop.circle")
            }
            .tag(AppTab.profile)
        }
        .tint(AppColors.brandPrimary)
    }
}
```

### 5.6 路由八条铁律

1. **严禁裸写魔术字符串跳转**：严禁在视图代码中手写 URL 字符串尝试做内部页面跳转，所有导航动作必须通过 `router.push(AppRoute.xxx)` 强类型枚举触发。
2. **Tab 路由独立隔离**：各个底导 Tab 必须拥有独立的 `NavigationStack` 与 `NavigationPath` 实例，严禁多 Tab 共享同一个全局栈，防止切换 Tab 时导航历史混乱。
3. **业务主键显式入参**：路由参数仅允许传递轻量基本类型（如 `deviceId: String`），严禁跨路由传递沉重的大对象或可变 Model 实体（详情页必须依据 ID 重新拉取或从二级本地缓存取数）。
4. **弹窗（Sheet / FullScreenCover）统一接管**：模态弹窗状态必须统一注册在 `AppRouter` 中管理，严禁在各个私有子组件中散落声明多个孤立的 `@State private var showSheet = false`。
5. **返回拦截防丢失**：针对包含表单输入的编辑类页面，必须使用 `.interactiveDismissDisabled(hasUnsavedChanges)` 阻止用户误滑下手势退出，并配合确认弹窗保护数据。
6. **转场性能保护**：自定义转场动画（`matchedGeometryEffect`）仅限用于核心产品展示页，常规业务页面必须遵从 iOS 原生平滑丝滑的系统级推入/弹出动效。
7. **DeepLink 统一解析调度**：外部 Scheme 或 Universal Link 唤醒必须由 `DeepLinkHandler` 统一拦截、校验鉴权状态后再映射为 `AppRoute`，严禁在各个 View 的 `onOpenURL` 中随意处理。
8. **路由监听与行为埋点解耦**：页面访问埋点与生命周期观测应通过 ViewModifier 或全局路由钩子统一记录，不污染业务视图的核心逻辑。

---

## 六、网络层（请求对象模式）

遵循开闭原则，所有 API 请求封装为继承自 `ApiRequest` 协议的独立强类型请求对象。底层 `URLSession` 调度器、请求拦截（统一签名头 `X-Signature` / `X-Nonce` / `Content-Digest` / `Idempotency-Key`）、并发安全的 Token 自动续期与通用响应外壳解包统一收敛在 `Core/Network`。

### 6.1 请求协议声明

```swift
// Core/Network/ApiRequest.swift
import Foundation

public enum HttpMethod: String, Sendable {
    case get = "GET"
    case post = "POST"
    case put = "PUT"
    case delete = "DELETE"
    case patch = "PATCH"
}

public protocol ApiRequest: Sendable {
    /// 接口响应返回的业务模型类型（必须遵循 Decodable & Sendable）
    associatedtype Response: Decodable, Sendable

    var path: String { get }
    var method: HttpMethod { get }
    var headers: [String: String]? { get }
    var queryItems: [URLQueryItem]? { get }
    var body: (any Encodable & Sendable)? { get }
    var requiresAuth: Bool { get }
    var timeoutInterval: TimeInterval { get }

    /// 默认解包由 ApiClient 统一执行，若接口有特殊反序列化需求可在此自定义
    func decode(from data: Data, decoder: JSONDecoder) throws -> Response
}

public extension ApiRequest {
    var method: HttpMethod { .post }
    var headers: [String: String]? { nil }
    var queryItems: [URLQueryItem]? { nil }
    var body: (any Encodable & Sendable)? { nil }
    var requiresAuth: Bool { true }
    var timeoutInterval: TimeInterval { 15.0 }

    func decode(from data: Data, decoder: JSONDecoder) throws -> Response {
        try decoder.decode(Response.self, from: data)
    }
}

private extension Encodable {
    func encoded(using encoder: JSONEncoder) throws -> Data {
        try encoder.encode(self)
    }
}
```

### 6.2 业务请求对象示例

```swift
// Features/Auth/Data/Requests/LoginRequest.swift
import Foundation

public struct LoginRequest: ApiRequest {
    public typealias Response = AuthToken

    public struct Payload: Encodable, Sendable {
        public let email: String
        public let password: String
    }

    public let payload: Payload

    public init(email: String, password: String) {
        self.payload = Payload(email: email, password: password)
    }

    public var path: String { "api/v1/auth/login" }
    public var method: HttpMethod { .post }
    public var requiresAuth: Bool { false }
    public var body: (any Encodable & Sendable)? { payload }
}
```

### 6.3 统一响应体外壳、拦截器与 ApiClient 调度

对齐《RESTful API 规范》与《统一错误码与异常处理规范》，字段命名采用 **`camelCase`**，时间戳采用 **RFC 3339 毫秒精度**（如 `2026-10-22T07:30:20.123Z`），业务成功统一为 **`code == 0`**：

```json
{
  "code": 0,
  "message": "success",
  "data": { ... },
  "traceId": "c3a1b098-9a74-4b21-a476-886ecb4f8d91",
  "timestamp": "2026-11-04T00:30:20.123Z",
  "details": [],
  "isRetryable": false
}
```

`ApiClient` 负责执行请求拦截管道、校验 HTTP 与业务状态码、利用 `actor` 单飞刷新 Token，并将纯净的 `data` 移交给模型反序列化：

```swift
// Core/Network/ApiResponseEnvelope.swift
import Foundation

public struct EmptyResponse: Decodable, Sendable, Equatable {
    public init() {}
}

public struct ApiErrorDetail: Decodable, Sendable, Equatable {
    public let field: String
    public let reason: String
}

struct ApiResponseEnvelope<T: Decodable>: Decodable {
    let code: Int
    let message: String
    let data: T?
    let traceId: String?
    let timestamp: String?
    let details: [ApiErrorDetail]?
    let isRetryable: Bool?
}

// Core/Network/Interceptors/NetworkInterceptor.swift
public protocol NetworkInterceptor: Sendable {
    func adapt(_ urlRequest: URLRequest, bodyData: Data?) async throws -> URLRequest
}

// Core/Network/AuthTokenCoordinator.swift
public actor AuthTokenCoordinator {
    private var refreshTask: Task<String, Error>?
    private let tokenStorage: KeychainStorageProtocol
    private let refreshHandler: @Sendable () async throws -> String

    public init(
        tokenStorage: KeychainStorageProtocol,
        refreshHandler: @escaping @Sendable () async throws -> String
    ) {
        self.tokenStorage = tokenStorage
        self.refreshHandler = refreshHandler
    }

    /// Single-Flight 并发合并：多个接口同时 401 时仅触发一次 Token 刷新
    public func refreshTokenIfNeeded() async throws -> String {
        if let ongoing = refreshTask {
            return try await ongoing.value
        }
        let task = Task {
            defer { refreshTask = nil }
            let newToken = try await refreshHandler()
            try await tokenStorage.save(newToken, for: .accessToken)
            return newToken
        }
        refreshTask = task
        return try await task.value
    }
}
```

```swift
// Core/Network/ApiClient.swift
import Foundation

public protocol ApiClientProtocol: Sendable {
    func send<R: ApiRequest>(_ request: R) async throws -> R.Response
}

public final class ApiClient: ApiClientProtocol {
    private let session: URLSession
    private let baseURL: URL
    private let tokenStorage: KeychainStorageProtocol
    private let tokenCoordinator: AuthTokenCoordinator?
    private let interceptors: [NetworkInterceptor]
    private let jsonDecoder: JSONDecoder
    private let jsonEncoder: JSONEncoder

    public init(
        baseURL: URL,
        tokenStorage: KeychainStorageProtocol,
        tokenCoordinator: AuthTokenCoordinator? = nil,
        interceptors: [NetworkInterceptor] = [],
        session: URLSession = .shared
    ) {
        self.baseURL = baseURL
        self.tokenStorage = tokenStorage
        self.tokenCoordinator = tokenCoordinator
        self.interceptors = interceptors
        self.session = session

        self.jsonDecoder = JSONDecoder()
        // 兼容带毫秒与不带毫秒的 RFC 3339 时间戳
        self.jsonDecoder.dateDecodingStrategy = .custom { decoder in
            let container = try decoder.singleValueContainer()
            let str = try container.decode(String.self)
            let withMillis = ISO8601DateFormatter()
            withMillis.formatOptions = [.withInternetDateTime, .withFractionalSeconds]
            if let date = withMillis.date(from: str) { return date }
            let standard = ISO8601DateFormatter()
            standard.formatOptions = [.withInternetDateTime]
            if let date = standard.date(from: str) { return date }
            throw DecodingError.dataCorruptedError(in: container, debugDescription: "Invalid ISO8601 date: \(str)")
        }

        self.jsonEncoder = JSONEncoder()
        self.jsonEncoder.dateEncodingStrategy = .iso8601
    }

    public func send<R: ApiRequest>(_ request: R) async throws -> R.Response {
        try await execute(request, allowTokenRefresh: true)
    }

    private func execute<R: ApiRequest>(_ request: R, allowTokenRefresh: Bool) async throws -> R.Response {
        let normalizedPath = request.path.trimmingCharacters(in: CharacterSet(charactersIn: "/"))
        var urlComponents = URLComponents(
            url: baseURL.appendingPathComponent(normalizedPath),
            resolvingAgainstBaseURL: true
        )
        urlComponents?.queryItems = request.queryItems

        guard let url = urlComponents?.url else {
            throw AppException.invalidURL
        }

        var urlRequest = URLRequest(url: url, timeoutInterval: request.timeoutInterval)
        urlRequest.httpMethod = request.method.rawValue
        urlRequest.setValue("application/json; charset=utf-8", forHTTPHeaderField: "Content-Type")
        urlRequest.setValue("application/json", forHTTPHeaderField: "Accept")
        request.headers?.forEach { urlRequest.setValue($1, forHTTPHeaderField: $0) }

        // 鉴权拦截（严格区分未登录与 Keychain 系统读取异常）
        if request.requiresAuth {
            let token: String?
            do {
                token = try await tokenStorage.get(for: .accessToken)
            } catch {
                throw AppException.unknown("Keychain read failed: \(error.localizedDescription)")
            }
            guard let token, !token.isEmpty else {
                throw AppException.unauthorized
            }
            urlRequest.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }

        // 编码强类型 Encodable Body
        var bodyData: Data?
        if let body = request.body {
            let data = try body.encoded(using: jsonEncoder)
            urlRequest.httpBody = data
            bodyData = data
        }

        // 执行拦截器（签名头、Content-Digest、Accept-Language、客户端版本信息等）
        for interceptor in interceptors {
            urlRequest = try await interceptor.adapt(urlRequest, bodyData: bodyData)
        }

        let (data, response): (Data, URLResponse)
        do {
            (data, response) = try await session.data(for: urlRequest)
        } catch let urlError as URLError {
            if urlError.code == .cancelled {
                throw AppException.cancelled
            } else if urlError.code == .timedOut {
                throw AppException.timeout
            } else {
                throw AppException.networkError(urlError.localizedDescription)
            }
        } catch {
            throw AppException.networkError(error.localizedDescription)
        }

        guard let httpResponse = response as? HTTPURLResponse else {
            throw AppException.invalidResponse
        }

        // 401 并发安全 Token 自动续期并重试一次
        if httpResponse.statusCode == 401, request.requiresAuth, allowTokenRefresh, let tokenCoordinator {
            _ = try await tokenCoordinator.refreshTokenIfNeeded()
            return try await execute(request, allowTokenRefresh: false)
        }

        // HTTP 非 2xx 时优先解析统一错误信封以保留 code / traceId / isRetryable
        guard (200...299).contains(httpResponse.statusCode) else {
            if httpResponse.statusCode == 401 {
                throw AppException.unauthorized
            }
            if httpResponse.statusCode == 403 {
                throw AppException.forbidden
            }
            if let envelope = try? jsonDecoder.decode(ApiResponseEnvelope<EmptyResponse>.self, from: data) {
                throw AppException.businessError(
                    code: envelope.code,
                    message: envelope.message,
                    traceId: envelope.traceId,
                    isRetryable: envelope.isRetryable ?? false
                )
            }
            throw AppException.serverError(code: httpResponse.statusCode, message: "HTTP Error \(httpResponse.statusCode)")
        }

        // 统一外壳解包（成功码 code == 0）
        do {
            let envelope = try jsonDecoder.decode(ApiResponseEnvelope<R.Response>.self, from: data)
            guard envelope.code == 0 else {
                throw AppException.businessError(
                    code: envelope.code,
                    message: envelope.message,
                    traceId: envelope.traceId,
                    isRetryable: envelope.isRetryable ?? false
                )
            }
            guard let resultData = envelope.data else {
                if let empty = EmptyResponse() as? R.Response {
                    return empty
                }
                throw AppException.parseError("Missing data payload")
            }
            return resultData
        } catch let appException as AppException {
            throw appException
        } catch {
            throw AppException.parseError(error.localizedDescription)
        }
    }
}
```

### 6.4 统一异常体系（`AppException`）

```swift
// Core/Error/AppException.swift
import Foundation

public enum AppException: Error, LocalizedError, Sendable, Equatable {
    case invalidURL
    case invalidResponse
    case networkError(String)
    case timeout
    case unauthorized
    case forbidden
    case serverError(code: Int, message: String)
    case businessError(code: Int, message: String, traceId: String? = nil, isRetryable: Bool = false)
    case parseError(String)
    case cancelled
    case unknown(String)

    public var traceId: String? {
        if case .businessError(_, _, let traceId, _) = self { return traceId }
        return nil
    }

    public var isRetryable: Bool {
        switch self {
        case .networkError, .timeout:
            return true
        case .businessError(_, _, _, let retryable):
            return retryable
        case .serverError(let code, _):
            return (500...599).contains(code) || code == 429
        case .invalidURL, .invalidResponse, .unauthorized, .forbidden, .parseError, .cancelled, .unknown:
            return false
        }
    }

    public var errorDescription: String? {
        switch self {
        case .invalidURL, .invalidResponse:
            return String(localized: "error_invalid_response", table: "Errors")
        case .networkError:
            return String(localized: "error_network_unavailable", table: "Errors")
        case .timeout:
            return String(localized: "error_timeout", table: "Errors")
        case .unauthorized:
            return String(localized: "error_unauthorized", table: "Errors")
        case .forbidden:
            return String(localized: "error_forbidden", table: "Errors")
        case .serverError(_, let msg), .businessError(_, let msg, _, _):
            return msg
        case .parseError:
            return String(localized: "error_parse_failed", table: "Errors")
        case .cancelled:
            return String(localized: "error_cancelled", table: "Errors")
        case .unknown(let msg):
            return msg
        }
    }
}
```

---

## 七、国际化方案（按 Feature 隔离的 String Catalogs）

项目全量基于 Xcode 15+ 官方 **String Catalogs (`.xcstrings`)** 体系（编译期自动生成兼容 **iOS 16+** 的本地化资源）。摒弃单一大文件合并冲突的痛点，推行 **按 Feature 物理隔离** 的国际化管理架构。

### 7.1 核心设计与优势

1. **Feature 物理隔离**：每个 Feature 拥有独立的 String Catalog 文件（如 `Features/Auth/Resources/Auth.xcstrings`、`Features/Home/Resources/Home.xcstrings`），修改文案互不干扰，彻底根除 Git 合并冲突。
2. **编译期强类型封装**：结合模块化的 `Text` 扩展或自动生成符号访问，杜绝魔术 Key 拼写错误。
3. **复数与动态格式化完备支持**：原生处理各类语言的复数变体（Zero / One / Few / Many / Other）与内联插值。

### 7.2 目录约定与引用范式

```
Features/
├── Auth/Resources/Auth.xcstrings       # 认证模块本地化
├── Home/Resources/Home.xcstrings       # 首页模块本地化
└── Profile/Resources/Profile.xcstrings # 个人中心本地化
Shared/Resources/
├── Common.xcstrings                    # 全局通用（确定、取消、重试、完成）
└── Errors.xcstrings                    # 全局错误兜底提示文案（对应 AppException）
```

在 View 中统一通过强类型扩展读取，文本与按钮标题保持一致：

```swift
// Shared/Extensions/Localization+Extensions.swift
import SwiftUI

extension Text {
    init(authKey key: String.LocalizationValue) {
        self.init(key, tableName: "Auth", bundle: .main)
    }

    init(commonKey key: String.LocalizationValue) {
        self.init(key, tableName: "Common", bundle: .main)
    }
}

// 界面调用
Text(authKey: "login_title")
Text(commonKey: "confirm")
```

### 7.3 国际化七条铁律

1. **严禁硬编码任何用户可见的自然语言文本**：所有 UI 界面上的按钮、标题、占位符、错误提示及系统权限说明（`InfoPlist.xcstrings`）必须存入对应的 `.xcstrings`。
2. **命名空间与前缀规范**：Key 一律采用小写下划线命名法（`snake_case`），并以模块或页面名为前缀，如 `login_title`、`device_offline_hint`。
3. **通用与错误文案集中下沉**："确定"、"取消"、"重试"、"保存"、"加载中" 放入 `Common.xcstrings`，通用网络/系统异常提示放入 `Errors.xcstrings`，严禁在各个 Feature 中重复定义。
4. **支持动态多语言切换**：应用内部提供语言切换选项时，不得仅依赖重启应用，必须通过 SwiftUI 视图树根节点的 `.environment(\.locale, currentLocale)` 触发全局平滑热切换。
5. **动态参数占位符严谨声明**：带参数的文案必须明确声明数据类型，例如：`"device_count" = "已连接 %lld 台设备"`。
6. **覆盖中英文双语 baseline**：每次提交代码必须确保 `en`（基准）与 `zh-Hans`（简体中文）双向 100% 翻译覆盖，未完成翻译禁止合并进主分支。
7. **L10n 一致性单元测试**：CI 构建中执行自动化脚本校验各个 String Catalog 的 Key 完整度，杜绝漏提或空字符。

---

## 八、UI 与交互规范

### 8.1 设计系统与 Design Tokens（`App/Theme/`）

全面对接《FUXIN CANRUI 智能照明 App 统一设计系统规范》，在 Swift 中将其固化为严格的设计令牌（Design Tokens）。

#### 8.1.1 色彩令牌体系（`AppColors.swift`）

```swift
// App/Theme/AppColors.swift
import SwiftUI

public enum AppColors {
    // 品牌色
    public static let brandPrimary = Color("BrandPrimary", bundle: .main)
    public static let brandSecondary = Color("BrandSecondary", bundle: .main)

    // 背景与表面
    public static let background = Color("AppBackground", bundle: .main)
    public static let surface = Color("AppSurface", bundle: .main)
    public static let surfaceSubtle = Color("AppSurfaceSubtle", bundle: .main)

    // 文本与图标
    public static let textPrimary = Color("TextPrimary", bundle: .main)
    public static let textSecondary = Color("TextSecondary", bundle: .main)
    public static let textDisabled = Color("TextDisabled", bundle: .main)

    // 边框与分割线
    public static let border = Color("BorderColor", bundle: .main)

    // 功能状态反馈色
    public static let success = Color("AppSuccess", bundle: .main)
    public static let warning = Color("AppWarning", bundle: .main)
    public static let error = Color("AppError", bundle: .main)
}
```

#### 8.1.2 间距系统（`AppSpacing.swift`）

基于 **4pt/8pt 黄金网格**，严禁在 View 中手写裸像素常量：

```swift
// App/Theme/AppSpacing.swift
import Foundation

public enum AppSpacing {
    public static let xxs: CGFloat = 2
    public static let xs: CGFloat = 4
    public static let sm: CGFloat = 8
    public static let md: CGFloat = 12
    public static let lg: CGFloat = 16  // 标准卡片内边距
    public static let xl: CGFloat = 20
    public static let xxl: CGFloat = 24
    public static let xxxl: CGFloat = 32

    public static let screenMargin: CGFloat = 16 // 屏幕两侧安全边距
}
```

#### 8.1.3 圆角系统（`AppRadius.swift`）

```swift
// App/Theme/AppRadius.swift
import Foundation

public enum AppRadius {
    public static let sm: CGFloat = 8      // 小型标签
    public static let md: CGFloat = 12     // 输入框、紧凑控制按键
    public static let card: CGFloat = 16   // 标准功能卡片
    public static let pill: CGFloat = 9999 // 胶囊全圆角
}
```

#### 8.1.4 排版系统（`AppTypography.swift`，iOS 16+ Dynamic Type）

严格对接系统 SF Pro 语义字阶（`Font.system(_:design:weight:)` 于 iOS 16.0 起支持），严禁写死固定字号，确保原生跟随系统 Dynamic Type 缩放；数值与读数类文案须附加 `.monospacedDigit()`：

```swift
// App/Theme/AppTypography.swift
import SwiftUI

public enum AppTypography {
    public static let largeTitle = Font.system(.largeTitle, design: .default, weight: .bold)
    public static let pageTitle = Font.system(.title, design: .default, weight: .bold)
    public static let sectionTitle = Font.system(.title3, design: .default, weight: .semibold)
    public static let cardTitle = Font.system(.headline, design: .default, weight: .semibold)
    public static let body = Font.system(.body, design: .default, weight: .regular)
    public static let bodyMedium = Font.system(.subheadline, design: .default, weight: .regular)
    public static let caption = Font.system(.caption, design: .default, weight: .regular)
}
```

### 8.2 统一用户交互与防御性编程

#### 8.2.1 触控热区防御（Apple HIG 44x44pt 铁律）
所有可点击控件（图标按钮、返回箭头、删除叉号）的视觉尺寸无论多小，其实际触摸感应区域必须至少满足 **44 × 44 pt**：
```swift
Button(action: onDismiss) {
    Image(systemName: "xmark")
        .font(.system(size: 16, weight: .medium))
        .foregroundStyle(AppColors.textSecondary)
        .frame(width: 44, height: 44) // 触控热区保证
        .contentShape(Rectangle())
}
```

#### 8.2.2 操作反馈与防抖防御
按钮在异步请求处理中必须自动切换为不可交互的 Loading 态，并统一通过 `HapticManager` 提供轻触感反馈（对齐《UI 规范》`.light` 标准，避免使用 iOS 17+ 专属 `.sensoryFeedback`）：
```swift
// Core/Utils/HapticManager.swift
import UIKit

@MainActor
public enum HapticManager {
    public static func impact(_ style: UIImpactFeedbackGenerator.FeedbackStyle = .light) {
        let generator = UIImpactFeedbackGenerator(style: style)
        generator.prepare()
        generator.impactOccurred()
    }
}
```

#### 8.2.3 键盘遮挡与滚动交互
表单界面必须保证：
1. 采用 `ScrollView` 包裹；
2. 开启 `.scrollDismissesKeyboard(.immediately)` 支持滑动即收起（iOS 16+ 原生）；
3. 点击页面任意空白处触发收起键盘（`dismissKeyboardOnTap()`）；
4. 密码项与关键数字项通过 `AppTextField(..., isSecure: true)` 提供一键清空与明密文切换。

#### 8.2.4 长文本截断与大字体布局防护
所有展示服务器返回文本的组件，必须显式声明 `.lineLimit(...)` 与截断方式，严禁因多行长文本撑破卡片或挤压邻近核心操作按钮：
```swift
Text(device.name)
    .font(AppTypography.cardTitle)
    .foregroundStyle(AppColors.textPrimary)
    .lineLimit(1)
    .truncationMode(.tail)
```

---

### 8.3 核心通用组件封装参考（`Shared/Components/`）

#### 1. 通用操作按钮 `AppButton`
```swift
// Shared/Components/AppButton.swift
import SwiftUI

public struct AppButton: View {
    public let title: String
    public var isLoading: Bool = false
    public var disabled: Bool = false
    public let action: () -> Void

    public init(
        title: String,
        isLoading: Bool = false,
        disabled: Bool = false,
        action: @escaping () -> Void
    ) {
        self.title = title
        self.isLoading = isLoading
        self.disabled = disabled
        self.action = action
    }

    public var body: some View {
        Button(action: {
            guard !isLoading && !disabled else { return }
            HapticManager.impact(.light)
            action()
        }) {
            ZStack {
                if isLoading {
                    ProgressView()
                        .tint(.white)
                } else {
                    Text(title)
                        .font(AppTypography.body.weight(.semibold))
                        .foregroundStyle(.white)
                }
            }
            .frame(maxWidth: .infinity)
            .frame(height: 48)
            .background(
                RoundedRectangle(cornerRadius: AppRadius.md, style: .continuous)
                    .fill(disabled ? AppColors.textDisabled : AppColors.brandPrimary)
            )
        }
        .disabled(disabled || isLoading)
        .animation(.easeInOut(duration: 0.2), value: isLoading)
        .animation(.easeInOut(duration: 0.2), value: disabled)
    }
}
```

#### 2. 通用四态闭环容器 `AsyncPhaseView` 与 iOS 16 空状态组件 `AppEmptyView`
```swift
// Shared/Components/AppEmptyView.swift
import SwiftUI

public struct AppEmptyView: View {
    public let title: String
    public let systemImage: String
    public var subtitle: String? = nil

    public var body: some View {
        VStack(spacing: AppSpacing.sm) {
            Image(systemName: systemImage)
                .font(.system(size: 44))
                .foregroundStyle(AppColors.textSecondary)
            Text(title)
                .font(AppTypography.cardTitle)
                .foregroundStyle(AppColors.textPrimary)
            if let subtitle {
                Text(subtitle)
                    .font(AppTypography.bodyMedium)
                    .foregroundStyle(AppColors.textSecondary)
                    .multilineTextAlignment(.center)
            }
        }
        .padding(AppSpacing.lg)
        .frame(maxWidth: .infinity, maxHeight: .infinity)
    }
}

// Shared/Components/AsyncPhaseView.swift
import SwiftUI

public struct AsyncPhaseView<T: Sendable, Content: View>: View {
    public let phase: AsyncPhase<T>
    public let onRetry: () -> Void
    @ViewBuilder public let content: (T) -> Content

    public var body: some View {
        switch phase {
        case .idle, .loading:
            VStack {
                Spacer()
                ProgressView()
                    .controlSize(.large)
                Spacer()
            }
            .frame(maxWidth: .infinity, maxHeight: .infinity)

        case .failure(let error):
            VStack(spacing: AppSpacing.md) {
                Spacer()
                Image(systemName: "wifi.exclamationmark")
                    .font(.system(size: 48))
                    .foregroundStyle(AppColors.textSecondary)

                Text(error.localizedDescription)
                    .font(AppTypography.body)
                    .foregroundStyle(AppColors.textSecondary)
                    .multilineTextAlignment(.center)
                    .padding(.horizontal, AppSpacing.lg)

                if let traceId = error.traceId {
                    Text("Trace ID: \(traceId)")
                        .font(AppTypography.caption)
                        .foregroundStyle(AppColors.textDisabled)
                        .textSelection(.enabled)
                }

                Button(String(localized: "retry", table: "Common"), action: onRetry)
                    .buttonStyle(.borderedProminent)
                    .tint(AppColors.brandPrimary)
                Spacer()
            }
            .frame(maxWidth: .infinity, maxHeight: .infinity)

        case .success(let data):
            content(data)
        }
    }
}
```

---

## 九、工程基建与质量规范

### 9.1 环境配置方案（`xcconfig` 集中映射）

严禁在 Swift 代码中写死接口 Host 与秘钥。工程设立 `Configurations/` 目录：

- `Debug.xcconfig`
- `Release.xcconfig`

在 Swift 中统一通过读取 `Bundle.main.infoDictionary` 映射为强类型配置：

```swift
// Core/Config/Environment.swift
import Foundation

public enum Environment {
    public static let apiBaseURL: URL = {
        guard let urlString = Bundle.main.object(forInfoDictionaryKey: "API_BASE_URL") as? String,
              let url = URL(string: urlString) else {
            fatalError("API_BASE_URL is not properly configured in xcconfig")
        }
        return url
    }()

    public static let isProduction: Bool = {
        #if DEBUG
        return false
        #else
        return true
        #endif
    }()
}
```

### 9.2 日志系统（`AppLogger`）

全面基于系统 `os.Logger`，彻底禁用 `print()`：

```swift
// Core/Logging/AppLogger.swift
import OSLog

public enum AppLogger {
    private static let subsystem = Bundle.main.bundleIdentifier ?? "com.fuxincanrui.app"

    public static let network = Logger(subsystem: subsystem, category: "Network")
    public static let auth = Logger(subsystem: subsystem, category: "Auth")
    public static let ui = Logger(subsystem: subsystem, category: "UI")
    public static let ble = Logger(subsystem: subsystem, category: "BLE")
}
```

调用范式：
```swift
AppLogger.network.info("Sending request: \(request.path, privacy: .public)")
AppLogger.auth.error("Login failed for user: \(email, privacy: .private)")
```

### 9.3 自动化测试规范（`Swift Testing`）

严禁在单元测试中使用 `Task.sleep` 赌异步回调时机；统一直接 `await` ViewModel 的 `async` 方法，杜绝 Flaky Test：

```swift
// Tests/Features/Auth/LoginViewModelTests.swift
import Testing
@testable import MyApp

@MainActor
struct LoginViewModelTests {
    @Test("验证邮箱与密码均有效时允许提交")
    func testValidationSucceedsWithValidCredentials() {
        let mockRepo = MockAuthRepository()
        let vm = LoginViewModel(authRepository: mockRepo)

        vm.email = "test@example.com"
        vm.password = "123456"

        #expect(vm.isInputValid == true)
    }

    @Test("验证登录成功后状态跃迁")
    func testLoginSuccessTransition() async {
        let mockRepo = MockAuthRepository()
        let expectedToken = AuthToken(accessToken: "mock_token")
        mockRepo.loginResult = .success(expectedToken)
        let vm = LoginViewModel(authRepository: mockRepo)

        vm.email = "test@example.com"
        vm.password = "123456"

        await vm.performLogin()

        #expect(vm.phase == .success(expectedToken))
        #expect(vm.phase.isLoading == false)
    }
}
```

### 9.4 代码质量门禁（`SwiftLint` + `swift-format`）

项目根目录维护 `.swiftlint.yml` 与 `.swift-format`，并接入 CI 流水线强制阻断：

1. **静态禁令规则（`custom_rules`）**：
   - 禁止生产代码出现 `print(`、`debugPrint(`；
   - 禁止出现硬编码颜色 `Color(red:` / `Color(hex:`；
   - 禁止在最低版本为 iOS 16.0 的工程中引入 `@Observable`、`@Bindable`、`SwiftData`、`ContentUnavailableView`。
2. **并发与复杂度门禁**：
   - 构建参数强制开启 `SWIFT_STRICT_CONCURRENCY = complete`；
   - CI 必须通过 `swiftlint --strict`（0 警告 / 0 错误）与 `xcodebuild test -destination 'platform=iOS Simulator,OS=16.4,name=iPhone 14'`（至少包含一台 iOS 16 模拟器验证兼容性）。

### 9.5 隐私合规与物联网通信基线（`PrivacyInfo.xcprivacy` & `Core/BLE`）

1. **Apple 隐私清单与权限本地化**：
   - 必须维护 `PrivacyInfo.xcprivacy`，如实声明 `UserDefaults`（`CA92.1`）等 Required Reason APIs 及网络/诊断采集项；
   - 蓝牙（`NSBluetoothAlwaysUsageDescription`）、本地网络（`NSLocalNetworkUsageDescription`）、相机扫码配网（`NSCameraUsageDescription`）权限文案统一放入 `InfoPlist.xcstrings`。
2. **BLE / MQTT 硬件通信层定位**：
   - `CoreBluetooth` 与 MQTT 会话统一封装在 `Core/BLE/` 与 `Core/MQTT/` 的专用 `actor` 或串行队列管理类内，向上层 Repository 暴露 `AsyncThrowingStream` 状态/帧数据流；
   - View 与 ViewModel 严禁直接 `import CoreBluetooth` 或持有 `CBPeripheral`。
3. **CoreData 并发隔离**：
   - 主线程读取使用 `viewContext`，所有批量缓存写入与后台同步必须在 `container.performBackgroundTask` 中执行，禁止跨线程传递 `NSManagedObject` 实例。

---

## 十、新增 Feature 的标准流程

1. **创建物理目录**：在 `Features/` 下新建 `<FeatureName>/`，包含 `Data/`（Requests、Models、Repository）、`Resources/`（`<FeatureName>.xcstrings`）。
2. **定义数据契约与仓储**：
   - 编写遵循 `ApiRequest` 协议的强类型 Request；
   - 声明不可变数据模型 `struct Model: Codable, Sendable`；
   - 声明 `XxxRepositoryProtocol` 并在 Repository 中接入 `ApiClient`。
3. **编写业务 ViewModel**：
   - 继承 `ObservableObject` 并标注 `@MainActor`；
   - 注入 Repository 协议实例；
   - 通过 `@Published private(set)` 暴露 `AsyncPhase<T>` 或只读状态。
4. **编写 SwiftUI 页面（Colocation）**：
   - 建立页面目录（如 `Features/<Feature>/DeviceDetail/`）；
   - 通过 `@StateObject` 持有 ViewModel，接入 `AppSpacing`、`AppColors`、`AppTypography`、`AppButton`；
   - 确保完整覆盖「加载中、空数据（`AppEmptyView`）、错误重试、成功渲染」四态闭环。
5. **注册强类型路由**：
   - 在 `App/Navigation/AppRoute.swift` 枚举中增加对应 Case（含 `id` 实现）；
   - 在 `AppRouteViewFactory` 的穷尽 `switch` 中关联对应目标页面。
6. **补齐多语言与测试**：
   - 在本模块专用的 `.xcstrings` 中录入中英文对应词条；
   - 在 `Tests/Features/<Feature>/` 中编写确定性 `async` ViewModel 状态跃迁单元测试。

---

## 十一、检查清单（Code Review 用）

- [ ] **最低版本兼容**：是否 100% 兼容 **iOS 16.0+**，未误用 `@Observable`、`SwiftData`、`ContentUnavailableView`、双参数 `.onChange` 等 iOS 17+ 专属 API？
- [ ] **架构合规**：View 中是否纯粹声明式渲染，绝无直接请求网络、修改 Repository 或编写业务逻辑的行为？
- [ ] **并发安全**：ViewModel 是否已声明 `@MainActor`？耗时任务是否放在非主线程或异步 `Task` 中？
- [ ] **不可变性**：数据模型是否为遵循 `Sendable` 的只读 `struct`？
- [ ] **取消机制**：耗时 `Task` 是否具备取消能力并在退出时调用？是否对 `Task.isCancelled` 做了防御检查？
- [ ] **触控与无障碍**：所有图标点击热区是否均达到 44 × 44 pt？是否通过语义字阶支持系统 Dynamic Type 字体缩放？
- [ ] **文本溢出防御**：动态服务器文本是否包裹了 `lineLimit` 与截断模式，是否能防撑破 UI？
- [ ] **设计系统对齐**：是否 100% 杜绝了硬编码魔法像素（必须采用 `AppSpacing`）与十六进制硬编码颜色（必须采用 `AppColors`）？
- [ ] **错误处理与本地化**：View 层是否仅通过 `AppException` 映射本地化文案并支持 `traceId` 展示？界面文案是否全部放入对应 `.xcstrings`？
- [ ] **日志合规**：生产代码中是否已彻底杜绝 `print()` 与 `debugPrint()`，统一采用 `AppLogger` 并对敏感数据脱敏？
- [ ] **质量门禁**：`SwiftLint` 0 警告 / 0 错误通过，iOS 16 模拟器单元测试全绿。

---

## 十二、AI 开发协作规范（AI_RULES 体系与落地）

### 12.1 核心架构定位

在 iOS 原生工程中，所有 AI 助手（Claude、Cursor、Copilot、Gemini、Antigravity）必须严格以工程根目录下的 `AI_RULES/RULES.md` 为统一总控清单。

### 12.2 十条绝对红线（Hard Bans）

1. **严禁引入 CocoaPods，且严禁私自添加未经批准的 SPM 第三方库**。
2. **严禁在 iOS 16.0+ 工程中使用 iOS 17+ 专属 API（如 `@Observable`、`@Bindable`、`SwiftData`、`ContentUnavailableView`）**，统一使用 `ObservableObject` + `@Published` + `@StateObject` 与 `CoreData`。
3. **严禁在 ViewModel / Repository 中 `import SwiftUI` 或引用任何 UIKit/UI 视图实例**。
4. **严禁在 View 的 `body` 渲染方法中直接触发改变状态或发起网络调用的副作用**。
5. **严禁裸写魔术字符串进行页面跳转**。
6. **严禁硬编码文本、硬编码颜色值与硬编码固定间距数值**。
7. **严禁在生产代码中使用 `print()`**，必须统一使用 `AppLogger`。
8. **严禁在 ViewModel 中省略 `@MainActor` 导致跨线程更新 UI 隐患**。
9. **严禁破坏 Feature 隔离直接跨模块引用私有类或视图**。
10. **严禁交付未通过 `SwiftLint` 静态检查或编译报错的代码**。
