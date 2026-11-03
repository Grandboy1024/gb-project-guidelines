## 一、技术选型

| 类别 | 选择 | 说明 |
| --- | --- | --- |
| 状态管理 | **Riverpod**（`flutter_riverpod` + `riverpod_annotation` + `riverpod_generator`） | 全项目唯一方案，Notifier 即 ViewModel |
| 架构模式 | **MVVM** | View / ViewModel / Repository / Service(DataSource) |
| 路由 | `go_router` | 声明式，配合 Riverpod 做登录重定向 |
| 网络 | `dio`，封装为请求对象模式 | 见第四节 |
| 数据类 / 序列化 | `freezed` + `json_serializable` | State、Model 全部不可变 |
| 本地存储 | `shared_preferences`（普通配置）+ `flutter_secure_storage`（token） | 需要结构化数据时再加 `drift` |
| 依赖注入 | **Riverpod 本身** | 不引入 get_it |
| 错误处理 | 自定义 `AppException` + `AsyncValue` | 不引入 Result / Either 库 |
| 国际化 | 官方 `flutter_localizations` + `intl` + gen-l10n（ARB），**按 feature 隔离** | 见第五节 |
| 日志 | `talker`（或 `logger`） | 统一入口，禁止 `print` |
| 测试 | `flutter_test` + `mocktail` |  |
| Lint | `very_good_analysis` | 开发初期就开启严格规则 |
| 环境配置 | `--dart-define-from-file` | dev / prod 两个 json |

版本一律使用当前 stable 最新版。代码生成命令：

bash

```bash
dart run build_runner build -d
```

---

## 二、目录结构

```
lib/
├── main.dart                     # 入口：只调用 bootstrap
├── bootstrap.dart                # 初始化(日志、存储、ProviderScope、错误捕获)
├── app/
│   ├── app.dart                  # MaterialApp.router
│   ├── router/
│   │   ├── app_router.dart       # go_router 配置(provider)
│   │   └── routes.dart           # 路由路径常量
│   ├── l10n/
│   │   └── delegates.dart        # 汇总各 feature 的 l10n delegate
│   └── theme/                    # light/dark、颜色、字体
├── core/                         # 与业务无关的基础设施
│   ├── config/env.dart           # 读取 dart-define
│   ├── network/
│   │   ├── api_client.dart       # dio 封装，唯一发请求的地方
│   │   ├── api_request.dart      # 请求基类
│   │   ├── interceptors/         # 公共头、签名、鉴权、日志
│   │   └── providers.dart
│   ├── storage/                  # kv 存储、secure 存储封装
│   ├── error/app_exception.dart
│   ├── logging/
│   └── utils/
├── shared/                       # 跨 feature 复用
│   ├── widgets/                  # 通用组件(无业务)
│   ├── extensions/               # context.l10n 等
│   └── providers/                # 全局 provider(locale、themeMode)
├── features/
│   └── auth/                     # 每个 feature 自成一体
│       ├── data/
│       │   ├── requests/         # LoginRequest 等请求对象
│       │   ├── models/           # freezed + json
│       │   └── auth_repository.dart
│       ├── domain/               # 可选，见规则 6
│       ├── l10n/                 # 该 feature 自己的文案
│       │   ├── auth_en.arb       # 模板语言，生成类 AuthLocalizations
│       │   ├── auth_zh.arb
│       │   └── generated/
│       └── presentation/         # 按页面组织，一个页面一个目录
│           ├── auth_l10n.dart    # context.authL10n 扩展
│           ├── login/
│           │   ├── login_page.dart          # View
│           │   ├── login_view_model.dart    # ViewModel
│           │   ├── login_state.dart         # State(freezed)
│           │   └── widgets/                 # 仅登录页使用的组件
│           ├── register/
│           │   ├── register_page.dart
│           │   ├── register_view_model.dart
│           │   ├── register_state.dart
│           │   └── widgets/
│           └── shared/           # 本 feature 内多个页面共用(widget、共享状态的 ViewModel)
└── l10n/                         # 全局公共文案(确定/取消/通用错误等)
    ├── app_en.arb                # 模板语言，生成类 AppLocalizations
    ├── app_zh.arb
    └── generated/

tool/
└── gen_l10n.sh                   # 一次生成全局 + 所有 feature 的 l10n
```

### 目录组织规则

1. **表现层按页面组织**：一个页面的 page、ViewModel、State、私有 widget 放在同一个目录，一个页面对应一个 ViewModel。
2. **同一 feature 内两个以上页面共用的内容放 `presentation/shared/`**；跨 feature 共用的放根目录 `shared/widgets/`。
3. **先放页面里，第二个页面真的需要时再提到 `shared/`**，不预先抽象。
4. **只有一个页面的 feature 保持扁平**：`presentation/` 下直接放 `xxx_page.dart`、`xxx_view_model.dart`、`xxx_state.dart` 和 `widgets/`；出现第二个页面时再拆成按页面的目录。
5. **`data/` 和 `l10n/` 不按页面拆**：Repository、请求对象被多个页面共用；ARB 以 feature 为粒度，避免生成类和 delegate 过碎。
6. **测试目录与源码镜像**，如 `test/features/auth/presentation/login/login_view_model_test.dart`。

---

## 三、MVVM 落地规则

### 3.1 角色对应

| 角色 | 实现 | 职责 |
| --- | --- | --- |
| **View** | `ConsumerWidget` / `ConsumerStatefulWidget` | 渲染、转发用户操作，不含业务逻辑 |
| **ViewModel** | `Notifier` / `AsyncNotifier` | 持有 UI 状态，调用 Repository，向 View 暴露不可变 State |
| **Model** | Repository + 数据模型 | Repository 是 ViewModel 唯一的数据入口 |

**依赖方向（单向）**：

```
View → ViewModel → Repository → ApiClient / Storage
```

禁止反向引用；View 不能直接访问 Repository 或 ApiClient。

### 3.2 示例：登录

**State**

dart

```dart
// features/auth/presentation/login/login_state.dartpart 'login_state.freezed.dart';@freezedabstract class LoginState with _$LoginState {  const factory LoginState({    @Default(false) bool submitting,    AppException? error,  }) = _LoginState;}
```

**ViewModel**

dart

```dart
// features/auth/presentation/login/login_view_model.dartpart 'login_view_model.g.dart';@riverpodclass LoginViewModel extends _$LoginViewModel {  @override  LoginState build() => const LoginState();  Future<bool> submit(String email, String password) async {    state = state.copyWith(submitting: true, error: null);    try {      await ref.read(authRepositoryProvider).login(email, password);      state = state.copyWith(submitting: false);      return true;    } on AppException catch (e) {      state = state.copyWith(submitting: false, error: e);      return false;    }  }}
```

**Repository**

dart

```dart
// features/auth/data/auth_repository.dartpart 'auth_repository.g.dart';@Riverpod(keepAlive: true)AuthRepository authRepository(Ref ref) =>    AuthRepository(ref.watch(apiClientProvider));class AuthRepository {  AuthRepository(this._client);  final ApiClient _client;  Future<AuthToken> login(String email, String password) =>      _client.send(LoginRequest(email: email, password: password));}
```

**View**

dart

```dart
class LoginPage extends ConsumerWidget {  const LoginPage({super.key});  @override  Widget build(BuildContext context, WidgetRef ref) {    final state = ref.watch(loginViewModelProvider);    final l10n = context.l10n;    return Scaffold(      body: FilledButton(        onPressed: state.submitting            ? null            : () => ref                .read(loginViewModelProvider.notifier)                .submit('a@b.com', '123456'),        child: Text(l10n.loginButton),      ),    );  }}
```

### 3.3 十条硬性规则

1. 页面级数据一律用 `@riverpod` 生成的 provider，不手写 `Provider(...)`。
2. State 必须是 freezed 不可变类；列表型异步数据用 `AsyncNotifier`，返回 `AsyncValue`。
3. ViewModel 中不出现 `BuildContext`、`Navigator`、`Widget`、`l10n`。导航和弹窗通过 View 里的 `ref.listen` 或方法返回值触发。
4. ViewModel 只暴露"错误类型"（`AppException`），错误文案在 View 中用 l10n 翻译。
5. Repository 只抛 `AppException`，不向上泄露 `DioException`。
6. `domain/` 默认不建。只有同一段业务逻辑被多个 ViewModel 复用时，才加 `domain/usecases/` 或 `entities/`。模型默认直接使用 `data/models` 里的 freezed 类，接口结构与 UI 需求明显不同时再拆 Entity。
7. feature 之间不互相 import。共用内容下沉到 `core/` 或 `shared/`；跨 feature 的数据通过 Repository provider 获取。
8. 全局单例（ApiClient、存储、Locale）做成 `keepAlive: true` 的 provider，测试时用 `ProviderScope(overrides: [...])` 替换。
9. 页面私有 widget 放该页面目录下的 `widgets/`；同一 feature 内多个页面共用的放 `presentation/shared/`；两个以上 feature 使用才移到根目录 `shared/widgets/`。
10. 文件命名：`xxx_page.dart`、`xxx_view_model.dart`、`xxx_state.dart`、`xxx_repository.dart`、`xxx_request.dart`，类名与文件名对应。

---

## 四、网络层（请求对象模式）

业务代码只继承基类、填写参数；公共头、签名、鉴权统一在 `core/network` 处理。

### 4.1 请求基类

dart

```dart
// core/network/api_request.dartenum HttpMethod { get, post, put, delete, patch }abstract class ApiRequest<T> {  const ApiRequest();  String get path;  HttpMethod get method => HttpMethod.post;  Map<String, dynamic>? get query => null;  Map<String, dynamic>? get body => null;  bool get requiresAuth => true;  /// 把响应 data 解析成业务对象  T parse(dynamic data);}
```

### 4.2 业务请求

dart

```dart
// features/auth/data/requests/login_request.dartclass LoginRequest extends ApiRequest<AuthToken> {  const LoginRequest({required this.email, required this.password});  final String email;  final String password;  @override  String get path => '/auth/login';  @override  bool get requiresAuth => false;  @override  Map<String, dynamic> get body => {'email': email, 'password': password};  @override  AuthToken parse(dynamic data) =>      AuthToken.fromJson(data as Map<String, dynamic>);}
```

### 4.3 ApiClient 职责

`ApiClient.send<T>(ApiRequest<T> req)` 负责：

- 拼装 URL、method、query、body。
- 拦截器依次处理：公共头 → 签名 → 附加 token（`requiresAuth` 为 true 时）→ 日志。
- 把 `DioException` 和业务错误码统一转换成 `AppException`，再调用 `req.parse`。
- 遇到 401 时尝试刷新 token，失败则通知 auth 状态登出。

### 4.4 异常体系

`AppException` 使用 sealed class，子类固定为：

- `NetworkException`
- `TimeoutException`
- `UnauthorizedException`
- `ServerException(code, message)`
- `ParseException`
- `UnknownException`

View 层可用 `switch` 做穷尽匹配。

---

## 五、国际化方案（按 feature 隔离）

### 5.1 总体思路

- **全局公共文案**：放在 `lib/l10n/`，生成类 `AppLocalizations`，只放通用内容（确定、取消、重试、通用错误、应用名等）。
- **feature 文案**：每个 feature 在自己的 `l10n/` 下维护 ARB，生成类 `XxxLocalizations`（如 `AuthLocalizations`）。gen-l10n 一次只生成一个类，所以按目录分别运行一次。
- 不采用"把各 feature 的 ARB 合并成一个"的做法，那样生成出来的类仍然是单一的，feature 之间的隔离会丢失。

好处：删除一个 feature 时，连同它的文案一起删掉，只需去掉 delegate 列表中的一行；多个 feature 并行开发时不会改同一个 ARB 文件。

### 5.2 配置

`pubspec.yaml`：

yaml

```yaml
dependencies:  flutter_localizations:    sdk: flutter  intl: any   # 版本跟随 flutter_localizations
```

- **不使用**根目录的 `l10n.yaml`，也不设置 `flutter: generate: true`；所有生成参数统一写在 `tool/gen_l10n.sh`，避免配置文件与命令行参数互相覆盖。
- 生成文件输出到源码目录（`generated/`），**提交进 git**。
- gen-l10n 的参数名以当前 stable 为准；若某个参数提示不存在，按 `flutter gen-l10n --help` 调整。

### 5.3 生成脚本

`tool/gen_l10n.sh`：

bash

```bash
#!/usr/bin/env bashset -e# 1) 全局公共flutter gen-l10n \  --arb-dir=lib/l10n \  --template-arb-file=app_en.arb \  --output-dir=lib/l10n/generated \  --output-localization-file=app_localizations.dart \  --output-class=AppLocalizations \  --no-nullable-getter# 2) 每个 feature（目录名 auth_profile -> 类名 AuthProfileLocalizations）for dir in lib/features/*/l10n; do  [ -d "$dir" ] || continue  feature=$(basename "$(dirname "$dir")")  cls=$(echo "$feature" | awk -F_ '{for(i=1;i<=NF;i++) printf toupper(substr($i,1,1)) substr($i,2)}')  flutter gen-l10n \    --arb-dir="$dir" \    --template-arb-file="${feature}_en.arb" \    --output-dir="$dir/generated" \    --output-localization-file="${feature}_localizations.dart" \    --output-class="${cls}Localizations" \    --no-nullable-getterdone
```

命名约定：feature 目录 `xxx` → ARB 文件 `xxx_en.arb`、`xxx_zh.arb` → 生成类 `XxxLocalizations`。

### 5.4 ARB 约定

全局（`lib/l10n/app_en.arb`）：

json

```json
{  "@@locale": "en",  "appName": "MyApp",  "cancel": "Cancel",  "retry": "Retry",  "errorNetwork": "Network error, please try again",  "errorUnauthorized": "Session expired, please sign in again",  "errorUnknown": "Something went wrong"}
```

feature（`features/auth/l10n/auth_en.arb`）：

json

```json
{  "@@locale": "en",  "loginButton": "Sign in",  "@loginButton": { "description": "登录按钮文案" },  "welcomeUser": "Welcome, {name}",  "@welcomeUser": { "placeholders": { "name": { "type": "String" } } },  "attemptsLeft": "{count, plural, =1{1 attempt left} other{{count} attempts left}}",  "@attemptsLeft": { "placeholders": { "count": { "type": "int" } } }}
```

- 模板语言用 `xxx_en.arb`，其他语言（`xxx_zh.arb` 等）按相同 key 补充。
- feature 内的 key **不加 feature 前缀**（类本身已隔离命名空间），使用 lowerCamelCase，如 `loginButton`。
- 复数、性别使用 ICU 语法，不在代码里手写判断。
- 日期、数字、货币使用 `intl` 的 `DateFormat` / `NumberFormat`，并传入当前 locale，不手拼字符串。

### 5.5 使用方式

dart

```dart
// shared/extensions/context_x.dart（全局）extension L10nX on BuildContext {  AppLocalizations get l10n => AppLocalizations.of(this);}// features/auth/presentation/auth_l10n.dart（feature 自己的）extension AuthL10n on BuildContext {  AuthLocalizations get authL10n => AuthLocalizations.of(this);}// 使用Text(context.authL10n.loginButton)   // feature 自己的Text(context.l10n.cancel)            // 全局公共的
```

`AppException` 到文案的映射放在全局，因为通用错误文案在全局 ARB 里：

dart

```dart
// shared/extensions/app_exception_x.dartextension AppExceptionL10n on AppException {  String message(AppLocalizations l10n) => switch (this) {        NetworkException() => l10n.errorNetwork,        UnauthorizedException() => l10n.errorUnauthorized,        ServerException(:final message) => message,        _ => l10n.errorUnknown,      };}
```

feature 特有的业务错误码，在该 feature 的 presentation 层用自己的 l10n 映射。

### 5.6 注册 delegate（app 层集中）

dart

```dart
// app/l10n/delegates.dartconst featureL10nDelegates = <LocalizationsDelegate<dynamic>>[  AuthLocalizations.delegate,  ProfileLocalizations.delegate,];// app/app.dartMaterialApp.router(  locale: ref.watch(localeControllerProvider),  localizationsDelegates: [    ...AppLocalizations.localizationsDelegates, // 含 Material / Cupertino / Widgets    ...featureL10nDelegates,  ],  supportedLocales: AppLocalizations.supportedLocales,  // ...)
```

app 层本来就允许依赖各 feature，所以这里的引用不违反依赖方向。

### 5.7 语言切换

dart

```dart
// shared/providers/locale_provider.dart@Riverpod(keepAlive: true)class LocaleController extends _$LocaleController {  @override  Locale? build() {    final code = ref.watch(prefsProvider).getString('locale');    return code == null ? null : Locale(code);   // null = 跟随系统  }  Future<void> set(Locale? locale) async {    state = locale;    // 持久化到 SharedPreferences，null 则删除  }}
```

### 5.8 一致性测试

所有 `l10n/` 目录必须提供相同的语言集合，用测试保证：

dart

```dart
// test/l10n_consistency_test.dartimport 'dart:convert';import 'dart:io';import 'package:flutter_test/flutter_test.dart';Set<String> _locales(Directory dir) => dir    .listSync()    .whereType<File>()    .where((f) => f.path.endsWith('.arb'))    .map((f) => jsonDecode(f.readAsStringSync())['@@locale'] as String)    .toSet();void main() {  test('各 l10n 目录语言集合一致', () {    final expected = _locales(Directory('lib/l10n'));    for (final d in Directory('lib/features').listSync().whereType<Directory>()) {      final l10n = Directory('${d.path}/l10n');      if (l10n.existsSync()) {        expect(_locales(l10n), expected, reason: l10n.path);      }    }  });}
```

### 5.9 规则

1. feature 自己的文案只放自己的 `l10n/`；全局 `l10n/` 只放通用文案，且**不引用任何 feature**。
2. feature 可以使用全局 `AppLocalizations`，但**不能使用其他 feature 的 l10n**（与"feature 之间不互相 import"一致）。
3. 某条文案被两个以上 feature 使用时，上移到全局 `l10n/`。
4. UI 中禁止硬编码中英文字符串，包括 `Text('...')`、`SnackBar`、`hintText` 等。
5. 后端返回的错误信息仅作兜底，优先用错误码映射到 l10n 文案。
6. 请求头统一携带 `Accept-Language`（取当前 locale），让后端也能返回对应语言。
7. 新增文案流程：改 `xxx_en.arb` → 补其他语言 → 运行 `tool/gen_l10n.sh`。
8. 新增语言：给全局和每个 feature 各加一个对应的 `_xx.arb`，一致性测试会提醒遗漏。
9. 新增 feature 的 l10n：建 `l10n/` 目录与 ARB → 运行脚本 → 在 `app/l10n/delegates.dart` 注册 delegate。

---

## 六、其他统一约定

- **启动流程**：`main.dart` → `bootstrap()`：初始化 binding → 读取 `Env` → 初始化 SharedPreferences 与日志 → 设置 `FlutterError.onError` 与 `runZonedGuarded` → `runApp(ProviderScope(overrides: [...], child: App()))`。
- **环境配置**：`env/dev.json`、`env/prod.json`，运行时使用 `flutter run --dart-define-from-file=env/dev.json`，在 `core/config/env.dart` 通过 `String.fromEnvironment` 读取。
- **路由**：路径常量放 `routes.dart`；`go_router` 的 `redirect` 监听 auth 状态 provider 实现登录拦截；页面参数用路径参数或 `extra`，不传 ViewModel。
- **主题**：`ColorScheme.fromSeed` + Material 3；颜色、间距、圆角常量集中在 `app/theme/`，页面中不写魔法数值。
- **测试**：
    - Repository：mock `ApiClient`。
    - ViewModel：`ProviderContainer(overrides: [authRepositoryProvider.overrideWithValue(mock)])`。
    - View：widget test，重点页面再做 golden test。
- **提交前检查**：`dart format .` → `flutter analyze` → `flutter test`，可加入 git hook。

---

## 七、新增 feature 的标准流程

1. 建立 `features/xxx/{data,presentation}`。
2. 在 `data/requests/` 编写请求对象，在 `data/models/` 编写 freezed 模型。
3. 编写 `xxx_repository.dart` 及其 provider。
4. 为每个页面建立 `presentation/<页面名>/` 目录，编写 State 与 ViewModel。
5. 在同一目录编写页面（私有组件放其 `widgets/`），并在 `app/router` 注册路由。
6. 在该 feature 的 `l10n/` 中添加 ARB（en、zh 等），运行 `dart run build_runner build -d` 与 `tool/gen_l10n.sh`，并在 `app/l10n/delegates.dart` 注册 delegate。

---

## 八、检查清单（Code Review 用）

- [ ]  View 里没有业务逻辑，没有直接访问 Repository / ApiClient
- [ ]  ViewModel 里没有 `BuildContext` / l10n
- [ ]  State 是 freezed 不可变类
- [ ]  Repository 只抛 `AppException`
- [ ]  没有硬编码用户可见字符串
- [ ]  feature 文案放在自己的 `l10n/`，没有塞进全局 ARB，也没有引用其他 feature 的 l10n
- [ ]  新 feature 的 l10n delegate 已在 `app/l10n/delegates.dart` 注册，各目录语言集合一致
- [ ]  feature 之间没有互相 import
- [ ]  表现层按页面组织：page、ViewModel、State 在同一目录，一个页面一个 ViewModel
- [ ]  只被一个页面使用的 widget 没有被提前放进 `shared/`
- [ ]  没有手写 `Provider(...)`，全部使用 `@riverpod`
- [ ]  没有 `print`，日志走统一入口
- [ ]  `dart format`、`flutter analyze`、`flutter test` 全部通过

## 一、技术选型

| 类别 | 选择 | 说明 |
| --- | --- | --- |
| 状态管理 | **Riverpod**（`flutter_riverpod` + `riverpod_annotation` + `riverpod_generator`） | 全项目唯一方案，Notifier 即 ViewModel |
| 架构模式 | **MVVM** | View / ViewModel / Repository / Service(DataSource) |
| 路由 | `go_router` | 声明式，配合 Riverpod 做登录重定向 |
| 网络 | `dio`，封装为请求对象模式 | 见第四节 |
| 数据类 / 序列化 | `freezed` + `json_serializable` | State、Model 全部不可变 |
| 本地存储 | `shared_preferences`（普通配置）+ `flutter_secure_storage`（token） | 需要结构化数据时再加 `drift` |
| 依赖注入 | **Riverpod 本身** | 不引入 get_it |
| 错误处理 | 自定义 `AppException` + `AsyncValue` | 不引入 Result / Either 库 |
| 国际化 | 官方 `flutter_localizations` + `intl` + gen-l10n（ARB），**按 feature 隔离** | 见第五节 |
| 日志 | `talker`（或 `logger`） | 统一入口，禁止 `print` |
| 测试 | `flutter_test` + `mocktail` |  |
| Lint | `very_good_analysis` | 开发初期就开启严格规则 |
| 环境配置 | `--dart-define-from-file` | dev / prod 两个 json |

版本一律使用当前 stable 最新版。代码生成命令：

```bash
dart run build_runner build -d
```

---

## 二、目录结构

```
lib/
├── main.dart                     # 入口：只调用 bootstrap
├── bootstrap.dart                # 初始化(日志、存储、ProviderScope、错误捕获)
├── app/
│   ├── app.dart                  # MaterialApp.router
│   ├── router/
│   │   ├── app_router.dart       # go_router 配置(provider)
│   │   └── routes.dart           # 路由路径常量
│   ├── l10n/
│   │   └── delegates.dart        # 汇总各 feature 的 l10n delegate
│   └── theme/                    # light/dark、颜色、字体
├── core/                         # 与业务无关的基础设施
│   ├── config/env.dart           # 读取 dart-define
│   ├── network/
│   │   ├── api_client.dart       # dio 封装，唯一发请求的地方
│   │   ├── api_request.dart      # 请求基类
│   │   ├── interceptors/         # 公共头、签名、鉴权、日志
│   │   └── providers.dart
│   ├── storage/                  # kv 存储、secure 存储封装
│   ├── error/app_exception.dart
│   ├── logging/
│   └── utils/
├── shared/                       # 跨 feature 复用
│   ├── widgets/                  # 通用组件(无业务)
│   ├── extensions/               # context.l10n 等
│   └── providers/                # 全局 provider(locale、themeMode)
├── features/
│   └── auth/                     # 每个 feature 自成一体
│       ├── data/
│       │   ├── requests/         # LoginRequest 等请求对象
│       │   ├── models/           # freezed + json
│       │   └── auth_repository.dart
│       ├── domain/               # 可选，见规则 6
│       ├── l10n/                 # 该 feature 自己的文案
│       │   ├── auth_en.arb       # 模板语言，生成类 AuthLocalizations
│       │   ├── auth_zh.arb
│       │   └── generated/
│       └── presentation/
│           ├── auth_l10n.dart    # context.authL10n 扩展
│           ├── pages/            # View：页面
│           ├── widgets/          # View：该 feature 私有组件
│           └── view_models/      # ViewModel + State
└── l10n/                         # 全局公共文案(确定/取消/通用错误等)
    ├── app_en.arb                # 模板语言，生成类 AppLocalizations
    ├── app_zh.arb
    └── generated/

tool/
└── gen_l10n.sh                   # 一次生成全局 + 所有 feature 的 l10n
```

---

## 三、MVVM 落地规则

### 3.1 角色对应

| 角色 | 实现 | 职责 |
| --- | --- | --- |
| **View** | `ConsumerWidget` / `ConsumerStatefulWidget` | 渲染、转发用户操作，不含业务逻辑 |
| **ViewModel** | `Notifier` / `AsyncNotifier` | 持有 UI 状态，调用 Repository，向 View 暴露不可变 State |
| **Model** | Repository + 数据模型 | Repository 是 ViewModel 唯一的数据入口 |

**依赖方向（单向）**：

```
View → ViewModel → Repository → ApiClient / Storage
```

禁止反向引用；View 不能直接访问 Repository 或 ApiClient。

### 3.2 示例：登录

**State**

```dart
// features/auth/presentation/view_models/login_state.dart
part 'login_state.freezed.dart';

@freezed
abstract class LoginState with _$LoginState {
  const factory LoginState({
    @Default(false) bool submitting,
    AppException? error,
  }) = _LoginState;
}
```

**ViewModel**

```dart
// features/auth/presentation/view_models/login_view_model.dart
part 'login_view_model.g.dart';

@riverpod
class LoginViewModel extends _$LoginViewModel {
  @override
  LoginState build() => const LoginState();

  Future<bool> submit(String email, String password) async {
    state = state.copyWith(submitting: true, error: null);
    try {
      await ref.read(authRepositoryProvider).login(email, password);
      state = state.copyWith(submitting: false);
      return true;
    } on AppException catch (e) {
      state = state.copyWith(submitting: false, error: e);
      return false;
    }
  }
}
```

**Repository**

```dart
// features/auth/data/auth_repository.dart
part 'auth_repository.g.dart';

@Riverpod(keepAlive: true)
AuthRepository authRepository(Ref ref) =>
    AuthRepository(ref.watch(apiClientProvider));

class AuthRepository {
  AuthRepository(this._client);
  final ApiClient _client;

  Future<AuthToken> login(String email, String password) =>
      _client.send(LoginRequest(email: email, password: password));
}
```

**View**

```dart
class LoginPage extends ConsumerWidget {
  const LoginPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(loginViewModelProvider);
    final l10n = context.l10n;
    return Scaffold(
      body: FilledButton(
        onPressed: state.submitting
            ? null
            : () => ref
                .read(loginViewModelProvider.notifier)
                .submit('a@b.com', '123456'),
        child: Text(l10n.loginButton),
      ),
    );
  }
}
```

### 3.3 十条硬性规则

1. 页面级数据一律用 `@riverpod` 生成的 provider，不手写 `Provider(...)`。
2. State 必须是 freezed 不可变类；列表型异步数据用 `AsyncNotifier`，返回 `AsyncValue`。
3. ViewModel 中不出现 `BuildContext`、`Navigator`、`Widget`、`l10n`。导航和弹窗通过 View 里的 `ref.listen` 或方法返回值触发。
4. ViewModel 只暴露"错误类型"（`AppException`），错误文案在 View 中用 l10n 翻译。
5. Repository 只抛 `AppException`，不向上泄露 `DioException`。
6. `domain/` 默认不建。只有同一段业务逻辑被多个 ViewModel 复用时，才加 `domain/usecases/` 或 `entities/`。模型默认直接使用 `data/models` 里的 freezed 类，接口结构与 UI 需求明显不同时再拆 Entity。
7. feature 之间不互相 import。共用内容下沉到 `core/` 或 `shared/`；跨 feature 的数据通过 Repository provider 获取。
8. 全局单例（ApiClient、存储、Locale）做成 `keepAlive: true` 的 provider，测试时用 `ProviderScope(overrides: [...])` 替换。
9. 页面私有 widget 放该 feature 的 `presentation/widgets/`；两个以上 feature 使用才移到 `shared/widgets/`。
10. 文件命名：`xxx_page.dart`、`xxx_view_model.dart`、`xxx_state.dart`、`xxx_repository.dart`、`xxx_request.dart`，类名与文件名对应。

---

## 四、网络层（请求对象模式）

业务代码只继承基类、填写参数；公共头、签名、鉴权统一在 `core/network` 处理。

### 4.1 请求基类

```dart
// core/network/api_request.dart
enum HttpMethod { get, post, put, delete, patch }

abstract class ApiRequest<T> {
  const ApiRequest();

  String get path;
  HttpMethod get method => HttpMethod.post;
  Map<String, dynamic>? get query => null;
  Map<String, dynamic>? get body => null;
  bool get requiresAuth => true;

  /// 把响应 data 解析成业务对象
  T parse(dynamic data);
}
```

### 4.2 业务请求

```dart
// features/auth/data/requests/login_request.dart
class LoginRequest extends ApiRequest<AuthToken> {
  const LoginRequest({required this.email, required this.password});
  final String email;
  final String password;

  @override
  String get path => '/auth/login';
  @override
  bool get requiresAuth => false;
  @override
  Map<String, dynamic> get body => {'email': email, 'password': password};
  @override
  AuthToken parse(dynamic data) =>
      AuthToken.fromJson(data as Map<String, dynamic>);
}
```

### 4.3 ApiClient 职责

`ApiClient.send<T>(ApiRequest<T> req)` 负责：

- 拼装 URL、method、query、body。
- 拦截器依次处理：公共头 → 签名 → 附加 token（`requiresAuth` 为 true 时）→ 日志。
- 把 `DioException` 和业务错误码统一转换成 `AppException`，再调用 `req.parse`。
- 遇到 401 时尝试刷新 token，失败则通知 auth 状态登出。

### 4.4 异常体系

`AppException` 使用 sealed class，子类固定为：

- `NetworkException`
- `TimeoutException`
- `UnauthorizedException`
- `ServerException(code, message)`
- `ParseException`
- `UnknownException`

View 层可用 `switch` 做穷尽匹配。

---

## 五、国际化方案（按 feature 隔离）

### 5.1 总体思路

- **全局公共文案**：放在 `lib/l10n/`，生成类 `AppLocalizations`，只放通用内容（确定、取消、重试、通用错误、应用名等）。
- **feature 文案**：每个 feature 在自己的 `l10n/` 下维护 ARB，生成类 `XxxLocalizations`（如 `AuthLocalizations`）。gen-l10n 一次只生成一个类，所以按目录分别运行一次。
- 不采用"把各 feature 的 ARB 合并成一个"的做法，那样生成出来的类仍然是单一的，feature 之间的隔离会丢失。

好处：删除一个 feature 时，连同它的文案一起删掉，只需去掉 delegate 列表中的一行；多个 feature 并行开发时不会改同一个 ARB 文件。

### 5.2 配置

`pubspec.yaml`：

```yaml
dependencies:
  flutter_localizations:
    sdk: flutter
  intl: any   # 版本跟随 flutter_localizations
```

- **不使用**根目录的 `l10n.yaml`，也不设置 `flutter: generate: true`；所有生成参数统一写在 `tool/gen_l10n.sh`，避免配置文件与命令行参数互相覆盖。
- 生成文件输出到源码目录（`generated/`），**提交进 git**。
- gen-l10n 的参数名以当前 stable 为准；若某个参数提示不存在，按 `flutter gen-l10n --help` 调整。

### 5.3 生成脚本

`tool/gen_l10n.sh`：

```bash
#!/usr/bin/env bash
set -e

# 1) 全局公共
flutter gen-l10n \
  --arb-dir=lib/l10n \
  --template-arb-file=app_en.arb \
  --output-dir=lib/l10n/generated \
  --output-localization-file=app_localizations.dart \
  --output-class=AppLocalizations \
  --no-nullable-getter

# 2) 每个 feature（目录名 auth_profile -> 类名 AuthProfileLocalizations）
for dir in lib/features/*/l10n; do
  [ -d "$dir" ] || continue
  feature=$(basename "$(dirname "$dir")")
  cls=$(echo "$feature" | awk -F_ '{for(i=1;i<=NF;i++) printf toupper(substr($i,1,1)) substr($i,2)}')
  flutter gen-l10n \
    --arb-dir="$dir" \
    --template-arb-file="${feature}_en.arb" \
    --output-dir="$dir/generated" \
    --output-localization-file="${feature}_localizations.dart" \
    --output-class="${cls}Localizations" \
    --no-nullable-getter
done
```

命名约定：feature 目录 `xxx` → ARB 文件 `xxx_en.arb`、`xxx_zh.arb` → 生成类 `XxxLocalizations`。

### 5.4 ARB 约定

全局（`lib/l10n/app_en.arb`）：

```json
{
  "@@locale": "en",
  "appName": "MyApp",
  "cancel": "Cancel",
  "retry": "Retry",
  "errorNetwork": "Network error, please try again",
  "errorUnauthorized": "Session expired, please sign in again",
  "errorUnknown": "Something went wrong"
}
```

feature（`features/auth/l10n/auth_en.arb`）：

```json
{
  "@@locale": "en",
  "loginButton": "Sign in",
  "@loginButton": { "description": "登录按钮文案" },
  "welcomeUser": "Welcome, {name}",
  "@welcomeUser": { "placeholders": { "name": { "type": "String" } } },
  "attemptsLeft": "{count, plural, =1{1 attempt left} other{{count} attempts left}}",
  "@attemptsLeft": { "placeholders": { "count": { "type": "int" } } }
}
```

- 模板语言用 `xxx_en.arb`，其他语言（`xxx_zh.arb` 等）按相同 key 补充。
- feature 内的 key **不加 feature 前缀**（类本身已隔离命名空间），使用 lowerCamelCase，如 `loginButton`。
- 复数、性别使用 ICU 语法，不在代码里手写判断。
- 日期、数字、货币使用 `intl` 的 `DateFormat` / `NumberFormat`，并传入当前 locale，不手拼字符串。

### 5.5 使用方式

```dart
// shared/extensions/context_x.dart（全局）
extension L10nX on BuildContext {
  AppLocalizations get l10n => AppLocalizations.of(this);
}

// features/auth/presentation/auth_l10n.dart（feature 自己的）
extension AuthL10n on BuildContext {
  AuthLocalizations get authL10n => AuthLocalizations.of(this);
}

// 使用
Text(context.authL10n.loginButton)   // feature 自己的
Text(context.l10n.cancel)            // 全局公共的
```

`AppException` 到文案的映射放在全局，因为通用错误文案在全局 ARB 里：

```dart
// shared/extensions/app_exception_x.dart
extension AppExceptionL10n on AppException {
  String message(AppLocalizations l10n) => switch (this) {
        NetworkException() => l10n.errorNetwork,
        UnauthorizedException() => l10n.errorUnauthorized,
        ServerException(:final message) => message,
        _ => l10n.errorUnknown,
      };
}
```

feature 特有的业务错误码，在该 feature 的 presentation 层用自己的 l10n 映射。

### 5.6 注册 delegate（app 层集中）

```dart
// app/l10n/delegates.dart
const featureL10nDelegates = <LocalizationsDelegate<dynamic>>[
  AuthLocalizations.delegate,
  ProfileLocalizations.delegate,
];

// app/app.dart
MaterialApp.router(
  locale: ref.watch(localeControllerProvider),
  localizationsDelegates: [
    ...AppLocalizations.localizationsDelegates, // 含 Material / Cupertino / Widgets
    ...featureL10nDelegates,
  ],
  supportedLocales: AppLocalizations.supportedLocales,
  // ...
)
```

app 层本来就允许依赖各 feature，所以这里的引用不违反依赖方向。

### 5.7 语言切换

```dart
// shared/providers/locale_provider.dart
@Riverpod(keepAlive: true)
class LocaleController extends _$LocaleController {
  @override
  Locale? build() {
    final code = ref.watch(prefsProvider).getString('locale');
    return code == null ? null : Locale(code);   // null = 跟随系统
  }

  Future<void> set(Locale? locale) async {
    state = locale;
    // 持久化到 SharedPreferences，null 则删除
  }
}
```

### 5.8 一致性测试

所有 `l10n/` 目录必须提供相同的语言集合，用测试保证：

```dart
// test/l10n_consistency_test.dart
import 'dart:convert';
import 'dart:io';
import 'package:flutter_test/flutter_test.dart';

Set<String> _locales(Directory dir) => dir
    .listSync()
    .whereType<File>()
    .where((f) => f.path.endsWith('.arb'))
    .map((f) => jsonDecode(f.readAsStringSync())['@@locale'] as String)
    .toSet();

void main() {
  test('各 l10n 目录语言集合一致', () {
    final expected = _locales(Directory('lib/l10n'));
    for (final d in Directory('lib/features').listSync().whereType<Directory>()) {
      final l10n = Directory('${d.path}/l10n');
      if (l10n.existsSync()) {
        expect(_locales(l10n), expected, reason: l10n.path);
      }
    }
  });
}
```

### 5.9 规则

1. feature 自己的文案只放自己的 `l10n/`；全局 `l10n/` 只放通用文案，且**不引用任何 feature**。
2. feature 可以使用全局 `AppLocalizations`，但**不能使用其他 feature 的 l10n**（与"feature 之间不互相 import"一致）。
3. 某条文案被两个以上 feature 使用时，上移到全局 `l10n/`。
4. UI 中禁止硬编码中英文字符串，包括 `Text('...')`、`SnackBar`、`hintText` 等。
5. 后端返回的错误信息仅作兜底，优先用错误码映射到 l10n 文案。
6. 请求头统一携带 `Accept-Language`（取当前 locale），让后端也能返回对应语言。
7. 新增文案流程：改 `xxx_en.arb` → 补其他语言 → 运行 `tool/gen_l10n.sh`。
8. 新增语言：给全局和每个 feature 各加一个对应的 `_xx.arb`，一致性测试会提醒遗漏。
9. 新增 feature 的 l10n：建 `l10n/` 目录与 ARB → 运行脚本 → 在 `app/l10n/delegates.dart` 注册 delegate。

---

## 六、其他统一约定

- **启动流程**：`main.dart` → `bootstrap()`：初始化 binding → 读取 `Env` → 初始化 SharedPreferences 与日志 → 设置 `FlutterError.onError` 与 `runZonedGuarded` → `runApp(ProviderScope(overrides: [...], child: App()))`。
- **环境配置**：`env/dev.json`、`env/prod.json`，运行时使用 `flutter run --dart-define-from-file=env/dev.json`，在 `core/config/env.dart` 通过 `String.fromEnvironment` 读取。
- **路由**：路径常量放 `routes.dart`；`go_router` 的 `redirect` 监听 auth 状态 provider 实现登录拦截；页面参数用路径参数或 `extra`，不传 ViewModel。
- **主题**：`ColorScheme.fromSeed` + Material 3；颜色、间距、圆角常量集中在 `app/theme/`，页面中不写魔法数值。
- **测试**：
    - Repository：mock `ApiClient`。
    - ViewModel：`ProviderContainer(overrides: [authRepositoryProvider.overrideWithValue(mock)])`。
    - View：widget test，重点页面再做 golden test。
- **提交前检查**：`dart format .` → `flutter analyze` → `flutter test`，可加入 git hook。

---

## 七、新增 feature 的标准流程

1. 建立 `features/xxx/{data,presentation}`。
2. 在 `data/requests/` 编写请求对象，在 `data/models/` 编写 freezed 模型。
3. 编写 `xxx_repository.dart` 及其 provider。
4. 在 `presentation/view_models/` 编写 State 与 ViewModel。
5. 在 `presentation/pages/` 编写页面，并在 `app/router` 注册路由。
6. 在该 feature 的 `l10n/` 中添加 ARB（en、zh 等），运行 `dart run build_runner build -d` 与 `tool/gen_l10n.sh`，并在 `app/l10n/delegates.dart` 注册 delegate。

---

## 八、检查清单（Code Review 用）

- [ ]  View 里没有业务逻辑，没有直接访问 Repository / ApiClient
- [ ]  ViewModel 里没有 `BuildContext` / l10n
- [ ]  State 是 freezed 不可变类
- [ ]  Repository 只抛 `AppException`
- [ ]  没有硬编码用户可见字符串
- [ ]  feature 文案放在自己的 `l10n/`，没有塞进全局 ARB，也没有引用其他 feature 的 l10n
- [ ]  新 feature 的 l10n delegate 已在 `app/l10n/delegates.dart` 注册，各目录语言集合一致
- [ ]  feature 之间没有互相 import
- [ ]  没有手写 `Provider(...)`，全部使用 `@riverpod`
- [ ]  没有 `print`，日志走统一入口
- [ ]  `dart format`、`flutter analyze`、`flutter test` 全部通过