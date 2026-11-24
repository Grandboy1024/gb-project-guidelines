# 架构决策记录 (DECISIONS)

规范编号：PROJECT-002  
版本：1.0.0  
适用范围：强制执行  

---

## ADR-001: 采用 Riverpod 2.x 代码生成作为唯一状态管理与依赖注入方案

- **决策状态**：已确认 (Confirmed)
- **背景背景**：Flutter 生态中存在多种状态管理方案（Bloc、Provider、GetX、get_it）。多方案混用会导致依赖混乱、生命周期失控以及维护心智负担成倍增加。
- **决策结论**：全项目统一采用 **Riverpod 2.x 代码生成体系**（`@riverpod`、`@Riverpod(keepAlive: true)`）作为唯一的全局状态管理与依赖注入 (DI) 中枢。
- **强制规则**：
  - 严禁手写旧版原生 Provider（如 `StateNotifierProvider`、`ChangeNotifierProvider`、`FutureProvider`）。
  - 严禁引入 `get_it` 或 `injectable`；所有实例装配与生命周期由 Riverpod 统一接管。
  - 页面级 Controller 默认 autoDispose；仅基础设施与全局会话允许声明 `keepAlive: true`。

---

## ADR-002: 采用 `go_router` 强类型代码生成路由方案

- **决策状态**：已确认 (Confirmed)
- **背景背景**：基于字符串拼装的动态路由缺乏编译期入参类型校验，重构时极易在运行期发生参数缺失崩溃；原生物理返回键在多 Tab 架构中难以维护导航栈。
- **决策结论**：统一采用官方推荐的 **`go_router` 搭配 `go_router_builder`** 代码生成作为全工程唯一路由框架。
- **强制规则**：
  - 严禁裸写魔术字符串跳转（如 `context.go('/orders/123')`）或使用旧版 `Navigator.push`。
  - 业务实体唯一主键必须声明为 Path 路径参数，保证外部 Universal Links 与 DeepLink 可达性。
  - `$extra` 仅作为转场骨架屏优化的可选项，接收页必须具备在 `$extra == null` 时从网络重新拉取数据的降级能力。
  - 底部多 Tab 结构必须使用 `StatefulShellRoute.indexedStack`。
  - Router 鉴权监听必须使用 `RouterRefreshNotifier` 与 `refreshListenable`，严禁在 Router Provider 中直接 `ref.watch(authControllerProvider)` 导致整树重建白屏。

---

## ADR-003: 采用请求对象模式 (`ApiRequest<T>`) 与响应外壳集中解包

- **决策状态**：已确认 (Confirmed)
- **背景背景**：随处手写 `dio.get(...)` 会导致公共请求头、取消令牌、错误捕获与响应解析逻辑散落各处，难以统一治理。
- **决策结论**：网络请求统一封装为继承自 `ApiRequest<T>` 的强类型请求对象；由 `ApiClient.send<T>()` 统一执行调度。
- **强制规则**：
  - 后端约定的标准通用响应体外壳（如 `{ "code": 200, "message": "...", "data": ... }`）由 `ApiClient` 统一验证与剥离。
  - 验证通过后，仅内层的纯业务 `data` 会被交付给 `req.parse(data)` 进行模型反序列化。
  - 底层网络异常统一拦截并转换为密封类体系 `AppException`。

---

## ADR-004: 采用按 Feature 物理隔离的 ARB 国际化与纯 Dart 跨平台构建

- **决策状态**：已确认 (Confirmed)
- **背景背景**：单一大文件 ARB 会引发高频代码合并冲突，且破坏 Feature 的可拔插独立性；依赖 bash 的合并脚本在 Windows 平台环境下经常执行失败。
- **决策结论**：每个 Feature 在自己的 `l10n/` 目录下独立维护专属 ARB；使用工程级跨平台纯 Dart 脚本 `tool/gen_l10n.dart` 一次性生成全局与全部 Feature 代码。
- **强制规则**：
  - 通用词条集中在 `lib/l10n/app_{en,zh}.arb`；模块专有词条留在模块内部。
  - Feature 之间禁止互相引用对方的本地化文案。
  - 各 Feature 的本地化 Delegate 统一在 `app/l10n/delegates.dart` 注册。

---

## ADR-005: 采用页面级扁平就近聚合目录架构 (Colocation)

- **决策状态**：已确认 (Confirmed)
- **背景背景**：传统嵌套极深的 `presentation/views/`、`presentation/controllers/` 导致开发者在修改单个页面时需要在深层目录树中频繁跳跃，开发体验差。
- **决策结论**：取消冗余包装层，将一个页面的 View（`xxx_page.dart`）、ViewModel（`xxx_controller.dart`）与仅其私有的子组件（`widgets/`）放置在同一目录下。
- **强制规则**：
  - 仅单个页面私有的组件放该页面私有 `widgets/`。
  - 模块内共享组件放 `features/<feature>/widgets/`。
  - 跨两个以上模块的通用组件才提升至 `shared/widgets/`。

---

## ADR-006: 采用 Design Tokens 驱动的统一设计系统

- **决策状态**：已确认 (Confirmed)
- **背景背景**：硬编码奇数像素和十六进制色值会造成界面杂乱，并阻碍暗黑模式与动态主题的扩展。
- **决策结论**：界面间距、圆角、色彩与字体排版全面基于 Design Tokens 驱动。
- **强制规则**：
  - 间距：一律使用 `AppSpacing` 常量（4/8 栅格步长）搭配 `Gap`。
  - 圆角：一律使用 `AppRadius` 常量。
  - 颜色：一律取自 `Theme.of(context).colorScheme` 或 `AppCustomColors` 语义色扩展。
  - 字体：一律使用 `Theme.of(context).textTheme` 并显式声明行高 `height`。

---

## ADR-007: 采用操作类异步控制器范式 (`AsyncNotifier<void>` + `AsyncValue.guard`)

- **决策状态**：已确认 (Confirmed)
- **背景背景**：为每个简单的表单提交或按钮点击手写包含 `submitting: bool`、`error: String?` 的不可变 State 类和配套 Freezed 生成，产生大量样板代码。
- **决策结论**：凡属于纯提交/修改/删除等操作，页面 Controller 优先继承 `AsyncNotifier<void>`，直接通过 `AsyncValue.guard` 托管 loading/error 状态。
- **强制规则**：
  - 仅在涉及复杂多字段前端联动缓存（如多级筛选器、多步骤向导）时才创建独立的不可变 State 类。

---

## ADR-008: 采用纯 Dart 集中环境配置 (`core/config/env.dart`) 并默认兜底 prod

- **决策状态**：已确认 (Confirmed)
- **背景背景**：依赖外部 `.env` 文件易因 CI/CD 环境文件缺失导致打包失败或误打包测试配置发布到生产环境。
- **决策结论**：统一在 `core/config/env.dart` 中使用 `String.fromEnvironment('ENV', defaultValue: 'prod')` 进行纯 Dart 静态配置。
- **强制规则**：
  - 默认无参打包永远指向生产环境 `prod`，杜绝测试接口泄漏风险。
  - 本地联调通过 `--dart-define=ENV=dev` 明确切换。
