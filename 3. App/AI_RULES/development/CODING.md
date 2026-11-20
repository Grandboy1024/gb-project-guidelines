# Dart & Flutter 编码与防御规范 (CODING)

规范编号：DEV-001  
版本：1.0.0  
适用范围：强制执行  

---

## 1. Modern Dart 3 语法规范

1. **穷尽模式匹配与 Switch 表达式**：
   - 针对密封类层级（如 `AppException`），必须使用 switch 表达式进行穷尽解包，严禁遗漏分支或写出脆弱的 `if-else`：
     ```dart
     final message = switch (exception) {
       NetworkException() => l10n.errorNetwork,
       TimeoutException() => l10n.errorTimeout,
       UnauthorizedException() => l10n.errorUnauthorized,
       ServerException(:final message) => message,
       _ => l10n.errorUnknown,
     };
     ```
2. **Records（元组）**：
   - 针对多值返回、轻量复合数据以及 Family Provider 的入参组合，优先使用 Dart 3 Records 替代冗余包装类：
     ```dart
     typedef PaginationParams = ({int page, int pageSize, String? query});
     ```
3. **默认不可变性（Immutability）**：
   - 所有状态类、领域模型、请求对象与常量配置必须保持完全不可变。
   - 业务数据模型统一使用 `@freezed` + `json_serializable` 生成。
   - 所有无内部可变状态的 Widget 与对象必须显式添加 `const` 构造函数。

---

## 2. 命名与文件组织约定

| 实体类别 | 命名风格 | 示范案例 |
|---|---|---|
| **文件名称** | `snake_case.dart` | `login_page.dart`、`auth_repository.dart`、`login_request.dart` |
| **类 / 扩展 / 枚举** | `PascalCase` | `LoginPage`、`AuthRepository`、`HttpMethod` |
| **方法 / 变量 / 入参** | `camelCase` | `fetchProducts()`、`userId`、`isLoading` |
| **私有成员** | `_camelCase` | `_client`、`_handleTap()` |
| **常量** | `lowerCamelCase` 或 `SCREAMING_SNAKE` | `appSpacingSm`、`kDefaultTimeout` |
| **Riverpod Provider** | 代码生成标准命名 | `loginControllerProvider`、`authRepositoryProvider` |
| **强类型路由类** | `XxxRoute` | `LoginRoute`、`OrderDetailRoute` |
| **本地化上下文扩展** | `XxxL10n` / `xxxL10n` | `context.authL10n`、`context.orderL10n` |

---

## 3. 页面就近聚合 (Colocation) 与结构约定

- **页面级就近聚合**：去掉了冗余的 `presentation/` 目录，将 View（`xxx_page.dart`）、ViewModel（`xxx_controller.dart`）和仅本页面使用的私有组件（`widgets/`）放置在同一层级：
  ```
  features/auth/login/
  ├── login_page.dart          # 视图 View (ConsumerWidget)
  ├── login_controller.dart    # 状态控制器 ViewModel (AsyncNotifier)
  └── widgets/                 # 仅供登录页私有的子组件
      └── login_header.dart
  ```
- **数据层集中共享**：整个 Feature 共享的数据请求、模型和仓储统一放置于 `features/<feature>/data/`。
- **国际化自闭环**：Feature 自身的 ARB 模版与上下文扩展统一放置于 `features/<feature>/l10n/`。

---

## 4. Riverpod 2.x 代码生成实践

1. **全面采用注解生成**：
   - 页面级控制器：`@riverpod class XxxController extends _$XxxController`（默认自动释放 autoDispose）。
   - 全局常驻单例 / 基础设施：`@Riverpod(keepAlive: true)`。
2. **异步提交优先采用 `AsyncNotifier<void>`**：
   - 针对登录、表单提交、修改、删除等操作，**严禁额外手写包含布尔值 `submitting: bool` 的 State 类**。统一采用 `AsyncNotifier<void>` 搭配 `AsyncValue.guard`：
     ```dart
     @riverpod
     class LoginController extends _$LoginController {
       @override
       FutureOr<void> build() {}

       Future<void> submit(String email, String password) async {
         state = const AsyncLoading();
         state = await AsyncValue.guard(
           () => ref.read(authRepositoryProvider).login(email, password),
         );
       }
     }
     ```
3. **Family Provider 参数等价性铁律**：
   - **至关重要**：Family 入参必须实现稳定的值等价性（重写 `==` 与 `hashCode`）。
   - 仅允许传入基本类型（`String`、`int`）、Dart 3 Records 或 `@freezed` 模型。
   - **严禁直接传入未重写等价性的裸 `List` 或 `Map`**，否则每次 `build` 都会被判定为新实例引发无限死循环网络请求。
4. **Widget 消费方式严格对齐**：
   - `build()` 内建立响应式依赖：`ref.watch(provider)` 或 `ref.watch(provider.select((s) => s.field))`。
   - 用户事件回调中读取实例：`ref.read(provider.notifier).submit()`。
   - 状态跃迁触发弹窗或路由跳转：在 `build()` 顶部使用 `ref.listen(provider, (prev, next) => ...)`。

---

---

## 5. UI 设计系统与 Design Tokens 规范（对应规范 8.1 节）

### 5.1 间距与圆角（AppSpacing & AppRadius）
- 界面 Padding、Margin 以及组件垂直/水平间隙，一律使用 `AppSpacing` 常量搭配 `Gap` 组件，**严禁出现未对齐的魔法数值**（如 7、11、13、17）：
  ```dart
  import 'package:gap/gap.dart';
  import '../../../app/theme/app_spacing.dart';

  // AppSpacing 步长：xs(4), sm(8), md(12), lg(16), xl(24), xxl(32), xxxl(48)
  const Gap(AppSpacing.md);
  padding: const EdgeInsets.all(AppSpacing.lg);
  ```
- 控件圆角统一使用 `AppRadius` 常量：
  ```dart
  // AppRadius 常量：sm(4), md(8), lg(12), xl(16), pill(999)
  borderRadius: BorderRadius.circular(AppRadius.md);
  ```

### 5.2 颜色与字体（ColorScheme & Typography）
- **严禁硬编码颜色**：严禁在 Widget 中手写 `Colors.white`、`Colors.black` 或十六进制色值（如 `Color(0xFF333333)`）。一律使用语义化角色：
  ```dart
  // 正确：
  final colorScheme = Theme.of(context).colorScheme;
  color: colorScheme.surface;
  color: colorScheme.primary;
  color: colorScheme.surfaceContainer;
  color: colorScheme.outline;
  color: context.colors.success; // AppCustomColors 业务语义色扩展

  // 严禁（违规）：
  color: Colors.white;
  color: Color(0xFF1E88E5);
  ```
- **严禁随意手写裸字号**：必须从 `Theme.of(context).textTheme` 取用语义化排版角色（`titleLarge`, `titleMedium`, `bodyLarge`, `bodyMedium`, `bodySmall`, `labelLarge` 等），且所有 TextStyle 必须显式声明规范行高 `height`（1.2 ~ 1.5），防止基线偏移：
  ```dart
  // 正确：
  style: Theme.of(context).textTheme.titleMedium;
  style: Theme.of(context).textTheme.bodySmall?.copyWith(color: colorScheme.onSurfaceVariant);

  // 严禁（违规）：
  style: TextStyle(fontSize: 15, color: Colors.black);
  ```

---

## 6. 统一用户交互规范（对应规范 8.2 节）

### 6.1 输入框与表单交互铁律
1. **快速清空按钮（Clear Button）**：所有单行文本输入框必须配备一键清除按钮（`suffixIcon`）。**交互细节**：仅在获得焦点且内容不为空时显现；点击清空后，**保持输入焦点不失焦**（软键盘不收起）。
2. **密码可见性切换**：密码输入框必须提供眼睛图标切换明文/密文（`obscureText`），切换时不改变已有光标位置。
3. **软键盘与滚动避让**：
   - 页面根部必须支持点击空白处收起键盘（通过手势收起聚焦）。
   - 包含表单的滚动容器统一配置 `keyboardDismissBehavior: ScrollViewKeyboardDismissBehavior.onDrag`。
   - 输入框页面使用 `SingleChildScrollView`，键盘弹出时自动将聚焦输入框滚动至可视区域上方。
4. **键盘动作键（TextInputAction）与焦点流转**：
   - 表单多输入框前序项设置 `TextInputAction.next`，最后一项设置 `TextInputAction.done` 或 `send`。
5. **搜索输入防抖（Debounce）**：
   - 实时输入搜索必须配备 300~500ms 防抖，并发起新请求前使用 `CancelToken` 取消上一次未决请求。

### 6.2 操作反馈与防重复点击
1. **异步提交按钮防重复点击（Loading Debounce）**：
   - 涉及网络请求的提交操作，在 `state.isLoading` 期间必须将 `onPressed` 设为 `null`（禁用态），防止重复点击。
   - 按钮内显示微型加载指示器（尺寸 18~20dp，保持与文字等高），严禁加载动画导致按钮尺寸突跳。
2. **状态反馈规范（SnackBars & Dialogs）**：
   - 轻量即时反馈（成功、通用网络异常）统一使用悬浮 `SnackBar`（`SnackBarBehavior.floating`），展示 2~3 秒自动消失，严禁打断用户链路。
   - 阻断性二次确认（不可逆高危操作）唤起 `AlertDialog`。左侧弱样式取消，右侧危险色确认。

### 6.3 触摸热区规范（Touch Target Size）
- **最小触摸热区 48×48 dp**：任何可交互控件的响应区域不得低于 48×48 逻辑像素（紧凑场景不低于 44×44）。
- 小尺寸图标（16~24dp）必须使用带内边距的 `IconButton` 或外层包裹 `Padding` 并配置 `behavior: HitTestBehavior.opaque`，杜绝按空。

### 6.4 列表常规手势与交互
1. **下拉刷新（Pull-to-Refresh）**：
   - 凡展示动态远端数据的列表页面必须支持下拉刷新（`RefreshIndicator`）。
   - 下拉刷新在后台静默替换数据，刷新失败仅弹 `SnackBar`，**严禁将已有列表清空跳成空白或全局错误页**。
2. **滑动操作与滑动删除（Swipe-to-Dismiss）**：
   - 不可逆高危删除**绝对禁止一滑即删**！必须通过 `confirmDismiss` 唤起确认弹窗，或在滑动后提供带「撤销（Undo）」按钮的 `SnackBar`。
3. **触控按压水波纹（Ink Splash）**：
   - 列表卡片与条目只要支持点击，必须包裹在 `InkWell`（配合 `Material`）或 `ListTile` 中，提供水波纹视觉反馈，严禁使用无反馈的空白 `GestureDetector`。

---

## 7. UI 边界条件与极端场景防御（对应规范 8.3 节）

### 7.1 长文本与排版防溢出
1. **Row / Flex 布局防溢出铁律**：
   - 在 `Row`、`Flex` 中放置 `Text` 时，若同级存在其他组件，该 `Text` **必须外层包裹 `Expanded` 或 `Flexible`**。
2. **单行文案截断**：
   - 用户昵称、卡片主标题必须显式指定 `maxLines: 1` 与 `overflow: TextOverflow.ellipsis`。
3. **不可断行特殊字符防爆**：
   - 连续无空格长英文、URL、哈希串设置 `softWrap: true` 或采用中间省略并提供复制功能。

### 7.2 系统大字体与无障碍缩放防护 (TextScaler Clamping)
- 在 `app.dart` 的 `MaterialApp.builder` 中，对 `MediaQuery` 的 `textScaler` 统一限幅（`clamp(minScaleFactor: 0.85, maxScaleFactor: 1.25)`），守住 UI 布局不崩塌。

### 7.3 全面屏与安全区适配 (SafeArea & Insets)
- 所有页面与自定义弹窗严格适配刘海屏、灵动岛与底部横条（Home Indicator）。
- 吸底组件（Bottom Bar / Bottom Sheet）必须使用 `SafeArea(top: false)` 或获取底部 insets，严禁按钮与系统手势区重叠。

### 7.4 页面与组件四态闭环 (Loading / Empty / Error / Success)
- 依赖异步请求的页面与列表，**严禁只有成功态和转圈态**，必须完整闭环：
  1. **加载中 (Loading)**：骨架屏或尺寸对齐的进度指示器。
  2. **空状态 (Empty)**：统一使用空状态插画 + 提示文案 + 引导按钮（如“去添加”或“刷新”）。
  3. **错误状态 (Error)**：人性化错误文案 + **明确的重试按钮**。
  4. **成功展示 (Success)**：正常渲染业务数据。

### 7.5 列表滚动与性能防御
1. **按需懒加载**：凡是动态长度列表一律使用 `ListView.builder`、`ListView.separated` 或 `SliverList` 视口复用；严禁使用 `SingleChildScrollView(Column(...))` 渲染未分页长列表。
2. **下拉刷新边界防爆**：外层包裹 `RefreshIndicator` 的列表必须配置 `physics: const AlwaysScrollableScrollPhysics()`，确保在初始空数据态或网络错误态依然可下拉刷新重试。
3. **滑动删除防崩溃**：`Dismissible` 的 `key` **必须使用强唯一业务实体 ID**（如 `Key('item_${item.id}')`），**绝对禁止使用 `ValueKey(index)`**；`onDismissed` 时必须同步从数据源移除该项，严禁数据源与 Widget 树脱节导致红屏报错。
4. **图片内存下采样**：网络图片统一传入 `memCacheWidth` / `memCacheHeight`，防止原图超高分辨率直接解码至内存导致 OOM。

---

## 8. 核心通用组件封装参考（对应规范 8.4 节）

业务页面优先直接使用 `shared/widgets/` 封装好的标准组件：
1. **通用输入框 `AppTextField`**：内置清空按钮与明密文切换、自动保持焦点与键盘避让。
2. **统一四态渲染组件 `AsyncValueWidget<T>`**：无缝映射 Riverpod 的 `AsyncValue`，自动处理 loading、error（带重试回调）与数据展示。
3. **列表左滑删除组件 `AppDismissibleItem`**：强实体 Key 校验、内置二次确认弹窗与错误色底。
