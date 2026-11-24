# 质量保证与测试规范 (TESTING)

规范编号：DEV-003  
版本：1.0.0  
适用范围：强制执行  

---

## 1. 测试体系与金字塔模型

本项目对核心业务逻辑、状态控制器与仓储层执行高标准的自动化测试覆盖：

```
        / \
       /   \      UI 交互验收测试 (SwiftUI View Testing)
      /-----\
     /       \    ViewModel 状态机与并发测试 (Swift Testing)
    /---------\
   /           \  Repository 仓储与 ApiClient 数据契约测试
  /-------------\
```

- **仓储层测试 (Repository Tests)**：Mock `ApiClientProtocol`，验证网络请求参数拼装、请求头注入以及外壳解包解析。
- **控制器测试 (ViewModel Tests)**：在 `@MainActor` 环境下运行，Mock 数据仓储，验证状态跃迁（初始、Loading、数据返回、AppException 异常捕获）与并发取消逻辑。
- **国际化一致性测试 (Localization Tests)**：自动化校验各个 Feature 的 `.xcstrings` 文件中英文覆盖率达到 100%。

---

## 2. 源码与测试目录镜像规范

测试文件必须严格镜像主工程目录结构放置在 `Tests/` 下：

```
Features/Auth/Login/LoginViewModel.swift
└── Tests/Features/Auth/LoginViewModelTests.swift

Features/Auth/Data/AuthRepository.swift
└── Tests/Features/Auth/AuthRepositoryTests.swift
```

---

## 3. 控制器单元测试标准范式 (`Swift Testing`)

```swift
import Testing
@testable import MyApp

@MainActor
@Suite("登录功能测试套件")
struct LoginViewModelTests {
    private let mockRepo = MockAuthRepository()
    
    @Test("验证邮箱与密码输入合法性校验")
    func testValidation() {
        let vm = LoginViewModel(authRepository: mockRepo)
        #expect(vm.isInputValid == false)
        
        vm.email = "test@domain.com"
        vm.password = "123456"
        #expect(vm.isInputValid == true)
    }
    
    @Test("验证网络异常时正确捕获并设置错误状态")
    func testNetworkErrorHandling() async {
        mockRepo.shouldFailWith = .networkError("Connection lost")
        let vm = LoginViewModel(authRepository: mockRepo)
        vm.email = "test@domain.com"
        vm.password = "123456"
        
        vm.login()
        try? await Task.sleep(nanoseconds: 30_000_000)
        
        #expect(vm.isLoading == false)
        #expect(vm.isSuccess == false)
        #expect(vm.errorMessage != nil)
    }
}
```
