# AI 统一规范清单 (Rules Manifest)

版本：1.1.0  
适用项目：iOS 原生移动端应用 (Native iOS App - Swift / SwiftUI, iOS 16.0+)  
最后更新：2026-11-25  

---

## 1. 规范定位与目的

本文档是本项目所有 AI 工具、智能助手（如 Cursor、Claude Code、GitHub Copilot、Gemini CLI、Antigravity、OpenDevin 等）的**唯一权威主入口**与规范索引清单。

所有 AI 在执行任何需求开发、缺陷修复、架构重构或测试编写任务前，**必须首先读取并严格遵守本清单及关联规范**。

本清单明确了各项规范的权威层级、任务适用矩阵以及冲突仲裁优先级。

---

## 2. 强制性核心规范 (Mandatory Core Rules)

以下文档对所有 AI 工具、所有开发场景均具有最高强制执行力：

| 规范编号 | 文件路径 | 核心内容概述 |
|---|---|---|
| **CORE-001** | [core/MASTER.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/3.%20App/3.2%20iOS/AI_RULES/core/MASTER.md) | AI 角色定位、行为准则与 10 条绝对红线禁令 |
| **CORE-002** | [core/WORKFLOW.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/3.%20App/3.2%20iOS/AI_RULES/core/WORKFLOW.md) | 5 阶段标准化作业流程 (SOP) 与任务执行规范 |
| **CORE-003** | [core/COMMUNICATION.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/3.%20App/3.2%20iOS/AI_RULES/core/COMMUNICATION.md) | 沟通规范、代码变更呈现与决策对齐机制 |

---

## 3. 开发规范 (Development Rules)

在进行 Swift 5.9+ / SwiftUI 代码编写、UI 还原、接口联调或测试时，必须严格遵守以下规范：

| 规范编号 | 文件路径 | 核心内容概述 |
|---|---|---|
| **DEV-001** | [development/CODING.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/3.%20App/3.2%20iOS/AI_RULES/development/CODING.md) | Modern Swift 严格并发、Design Tokens、44pt 触控防御与 UI 边界防护 |
| **DEV-002** | [development/ARCHITECTURE.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/3.%20App/3.2%20iOS/AI_RULES/development/ARCHITECTURE.md) | Clean MVVM、`ObservableObject` + `@Published` 状态管理、`NavigationStack` 强类型路由与 `ApiRequest` 模式 |
| **DEV-003** | [development/TESTING.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/3.%20App/3.2%20iOS/AI_RULES/development/TESTING.md) | Swift Testing 现代断言、异步并发测试、Protocol Mock 仓储测试与 String Catalogs 一致性 |

---

## 4. 项目知识库 (Project Knowledge)

以下文档提供项目的业务全景、历史技术决策与当前进展：

| 规范编号 | 文件路径 | 核心内容概述 |
|---|---|---|
| **PROJECT-001** | [project/CONTEXT.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/3.%20App/3.2%20iOS/AI_RULES/project/CONTEXT.md) | 业务背景、产品定位与核心功能模块划分 |
| **PROJECT-002** | [project/DECISIONS.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/3.%20App/3.2%20iOS/AI_RULES/project/DECISIONS.md) | 已确认的技术架构决策 (ADR-001 ~ ADR-006) |
| **PROJECT-003** | [project/STATUS.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/3.%20App/3.2%20iOS/AI_RULES/project/STATUS.md) | 项目当前进展阶段、已完成模块与后续演进目标 |

---

## 5. 核心技术选型速查表

AI 生成的所有代码必须 100% 契合以下技术决策，严禁违背：

| 技术领域 | 选型方案 | 对应规范索引 |
|---|---|---|
| **最低支持版本** | **iOS 16.0+**（禁用 `@Observable`、`SwiftData`、`ContentUnavailableView` 等 iOS 17+ 专属 API） | `DECISIONS.md` / `iOS 项目规范.md` 第一节 |
| **语言与并发** | Swift 5.9+ / Swift 6，Strict Concurrency Complete，`async`/`await` + `Task` + `@MainActor` | `DEV-001` / `iOS 项目规范.md` 第一节 |
| **架构模式** | Clean MVVM + 页面就近聚合 (Colocation) | `DEV-002` / `iOS 项目规范.md` 第三节 |
| **状态管理** | `ObservableObject` + `@Published`（`@StateObject` / `@ObservedObject` / `@EnvironmentObject`） | `DEV-002` / `iOS 项目规范.md` 第四节 |
| **路由导航** | `NavigationStack` + `NavigationPath` + 强类型 `AppRoute` 枚举 | `DEV-002` / `iOS 项目规范.md` 第五节 |
| **网络层** | `URLSession` + 请求对象模式 (`ApiRequest<T>`) + 统一响应外壳解包 | `DEV-002` / `iOS 项目规范.md` 第六节 |
| **数据类 / 序列化** | 不可变 `struct` 遵循 `Codable` & `Sendable` | `DEV-001` / `iOS 项目规范.md` 第一节 |
| **UI 与设计系统** | SwiftUI + HIG + `AppSpacing` + `AppRadius` + `AppColors` 语义色 | `DEV-001` / `iOS 项目规范.md` 第八节 |
| **国际化方案** | 按 Feature 隔离的 String Catalogs (`.xcstrings`) + 强类型访问扩展 | `DEV-002` / `iOS 项目规范.md` 第七节 |
| **环境配置** | `xcconfig` 映射集中配置 (`Core/Config/Environment.swift`) | `DEV-002` / `iOS 项目规范.md` 第九节 |
| **日志框架** | `os.Logger` 统一子系统与分类，严禁裸用 `print()` | `DEV-001` / `iOS 项目规范.md` 第九节 |
| **测试框架** | `Swift Testing` (`@Test`, `#expect`) + `XCTest` | `DEV-003` / `iOS 项目规范.md` 第九节 |
| **依赖包管理** | 原生 **SPM** (`Package.swift` / `Package.resolved`)，禁止引入 CocoaPods | `DECISIONS.md (ADR-006)` / `iOS 项目规范.md` 第一节 |

---

## 6. 执行准则与约束

1. **先读后写**：在编写或修改任何代码前，必须先查阅本清单并定位具体规范。
2. **严格守约**：不得因便利性擅自简化或绕过架构约束。
3. **禁止擅自重新定义**：未经开发者明确指令，严禁自行推翻既有技术选型或引入第三方框架。
4. **并发闭环**：UI 交互与 ViewModel 必须确保主线程隔离（`@MainActor`），异步并发必须处理 `Task.isCancelled`。
5. **及时报告缺失**：若发现引用的规范文件不存在，必须第一时间向开发者报告。

---

## 7. 文档冲突优先级

当不同指令或文档出现冲突时，严格按照以下优先级从高到低仲裁：

1. 系统与平台安全、运行环境与沙箱边界约束。
2. 开发者针对当前任务给出的显式指令。
3. 已确认的架构决策记录 (`project/DECISIONS.md`)。
4. 核心行为准则与红线 (`core/MASTER.md`, `core/WORKFLOW.md`)。
5. 任务相关的开发规范 (`development/CODING.md`, `ARCHITECTURE.md`, `TESTING.md`)。
6. 项目背景与状态记录 (`project/CONTEXT.md`, `STATUS.md`)。

---

## 8. 规范维护机制

本文档是项目规范的最高登记处。
当新增、移除或调整任何规范文档时，必须同步更新本清单并在 [CHANGELOG.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/3.%20App/3.2%20iOS/AI_RULES/CHANGELOG.md) 中记录版本演进。
