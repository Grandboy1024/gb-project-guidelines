# 质量保证与测试规范 (TESTING)

规范编号：DEV-003  
版本：1.0.0  
适用范围：强制执行  

---

## 1. 测试体系与金字塔模型

本项目对核心业务逻辑、状态控制器与公共组件执行高标准的自动化测试覆盖：

```
        / \
       /   \      Golden / 关键页面端到端测试
      /-----\
     /       \    Widget 视图与交互测试
    /---------\
   /           \  Unit 单元测试 (Controller 状态机 & Repository 适配)
  /-------------\
```

- **仓储层测试 (Repository Tests)**：Mock `ApiClient`，验证网络请求参数拼装、取消令牌传递以及响应外壳解包解析。
- **控制器测试 (Controller Tests)**：使用独立的 `ProviderContainer`，Mock 数据仓储，验证状态跃迁（初始、Loading、数据返回、AppException 异常捕获）与乐观更新回滚。
- **视图层测试 (Widget Tests)**：验证组件在不同状态下的渲染、空状态引导按钮、长文本截断、输入框交互及点击事件。
- **国际化一致性测试 (L10n Tests)**：自动校验全局与各个 Feature 的 ARB 文件语言集合保持严格一致。

---

## 2. 源码与测试目录镜像规范

测试文件必须严格镜像 `lib/` 目录结构放置在 `test/` 下：

```
lib/features/auth/login/login_controller.dart
└── test/features/auth/login/login_controller_test.dart

lib/features/auth/data/auth_repository.dart
└── test/features/auth/data/auth_repository_test.dart
```

---

## 3. 控制器单元测试标准范式 (`ProviderContainer`)

测试 Controller 必须使用 `ProviderContainer`，通过 `overrideWithValue` 替换依赖，严禁在测试中发起真实网络请求：

```dart
// test/features/auth/login/login_controller_test.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:my_app/core/error/app_exception.dart';
import 'package:my_app/features/auth/data/auth_repository.dart';
import 'package:my_app/features/auth/data/models/auth_token.dart';
import 'package:my_app/features/auth/login/login_controller.dart';

class MockAuthRepository extends Mock implements AuthRepository {}

void main() {
  late MockAuthRepository mockAuthRepository;
  late ProviderContainer container;

  setUp(() {
    mockAuthRepository = MockAuthRepository();
    container = ProviderContainer(
      overrides: [
        authRepositoryProvider.overrideWithValue(mockAuthRepository),
      ],
    );
  });

  tearDown(() => container.dispose());

  test('初始状态应为 AsyncData(null)', () {
    final state = container.read(loginControllerProvider);
    expect(state, const AsyncData<void>(null));
  });

  test('submit 提交成功时应正确经历 loading 并回到 AsyncData', () async {
    when(() => mockAuthRepository.login('test@test.com', 'password123'))
        .thenAnswer((_) async => const AuthToken(accessToken: 'token123'));

    final controller = container.read(loginControllerProvider.notifier);
    final future = controller.submit('test@test.com', 'password123');

    // 验证 loading 状态
    expect(container.read(loginControllerProvider).isLoading, true);

    await future;

    // 验证成功结束状态
    final state = container.read(loginControllerProvider);
    expect(state.hasError, false);
    expect(state.isLoading, false);
  });

  test('submit 失败时应正确捕获 AppException 并转为 AsyncError', () async {
    when(() => mockAuthRepository.login(any(), any()))
        .thenThrow(const NetworkException('网络连接超时'));

    final controller = container.read(loginControllerProvider.notifier);
    await controller.submit('test@test.com', 'wrong');

    final state = container.read(loginControllerProvider);
    expect(state.hasError, true);
    expect(state.error, isA<NetworkException>());
  });
}
```

---

## 4. 仓储层单元测试范式 (`Mocktail`)

```dart
// test/features/auth/data/auth_repository_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:my_app/core/network/api_client.dart';
import 'package:my_app/features/auth/data/auth_repository.dart';
import 'package:my_app/features/auth/data/models/auth_token.dart';
import 'package:my_app/features/auth/data/requests/login_request.dart';

class MockApiClient extends Mock implements ApiClient {}

void main() {
  late MockApiClient mockClient;
  late AuthRepository repository;

  setUpAll(() {
    registerFallbackValue(const LoginRequest(email: '', password: ''));
  });

  setUp(() {
    mockClient = MockApiClient();
    repository = AuthRepository(mockClient);
  });

  test('login 正确组装 LoginRequest 并解析返回的 AuthToken', () async {
    const expectedToken = AuthToken(accessToken: 'abc123456');
    when(() => mockClient.send<AuthToken>(any()))
        .thenAnswer((_) async => expectedToken);

    final result = await repository.login('test@example.com', 'pwd');
    expect(result, expectedToken);

    verify(() => mockClient.send<AuthToken>(any(
      that: isA<LoginRequest>()
          .having((r) => r.email, 'email', 'test@example.com')
          .having((r) => r.password, 'password', 'pwd'),
    ))).called(1);
  });
}
```

---

## 5. 国际化多语言一致性测试

确保所有 Feature 的 ARB 目录与全局公共 ARB 保持完全一致的语言环境支持：

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
  test('各 Feature 的 l10n 目录语言集合与全局 l10n 保持严格一致', () {
    final expected = _locales(Directory('lib/l10n'));
    final featuresDir = Directory('lib/features');
    if (!featuresDir.existsSync()) return;

    for (final d in featuresDir.listSync().whereType<Directory>()) {
      final l10n = Directory('${d.path}/l10n');
      if (l10n.existsSync()) {
        expect(_locales(l10n), expected, reason: '语言环境缺失或不匹配: ${l10n.path}');
      }
    }
  });
}
```

---

## 6. 本地提交与 CI 质量门禁套件

任何代码提交或任务交付前，必须在工程根目录下执行并通过以下三道质量门禁：

```bash
# 1. 检查代码格式是否完全合规
dart format --set-exit-if-changed .

# 2. 静态代码分析，零警告与零错误
flutter analyze

# 3. 运行全量自动化测试套件
flutter test
```
