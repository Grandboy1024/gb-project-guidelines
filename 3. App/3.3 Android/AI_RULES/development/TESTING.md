# 质量保证与测试规范 (TESTING)

规范编号：DEV-003  
版本：1.0.0  
适用范围：强制执行  

---

## 1. 测试体系与金字塔模型

本项目对核心业务逻辑、状态控制器与仓储层执行高标准的自动化测试覆盖：

```
        / \
       /   \      UI 交互验收测试 (ComposeTestRule)
      /-----\
     /       \    ViewModel 状态机与协程测试 (JUnit 5 + Turbine)
    /---------\
   /           \  Repository 仓储与 ApiClient 数据契约测试 (MockK)
  /-------------\
```

- **仓储层测试 (Repository Tests)**：Mock `ApiClient`，验证网络请求参数拼装、请求头注入以及外壳解包解析。
- **控制器测试 (ViewModel Tests)**：在 `StandardTestDispatcher` 环境下运行，Mock 数据仓储，使用 `Turbine` 验证状态流跃迁（初始、Loading、数据返回、AppException 异常捕获）。
- **国际化一致性测试 (Localization Tests)**：自动化校验各个 Feature 的 `strings.xml` 中英文覆盖率达到 100%。

---

## 2. 源码与测试目录镜像规范

测试文件必须严格镜像主工程目录结构放置在 `test/` 下：

```
app/src/main/java/com/fuxincanrui/app/features/auth/login/LoginViewModel.kt
└── app/src/test/java/com/fuxincanrui/app/features/auth/LoginViewModelTest.kt

app/src/main/java/com/fuxincanrui/app/features/auth/data/repository/AuthRepository.kt
└── app/src/test/java/com/fuxincanrui/app/features/auth/AuthRepositoryTest.kt
```

---

## 3. 控制器单元测试标准范式 (`Turbine` + `MockK`)

```kotlin
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
    fun `verify input validation logic`() {
        assertFalse(viewModel.uiState.value.isInputValid)

        viewModel.onAction(LoginUiAction.EmailChanged("test@test.com"))
        viewModel.onAction(LoginUiAction.PasswordChanged("123456"))

        assertTrue(viewModel.uiState.value.isInputValid)
    }

    @Test
    fun `verify state transition during successful login`() = runTest(testDispatcher) {
        coEvery { repository.login(any(), any()) } returns AuthToken("mock_token")

        viewModel.onAction(LoginUiAction.EmailChanged("test@test.com"))
        viewModel.onAction(LoginUiAction.PasswordChanged("123456"))

        viewModel.uiState.test {
            val initial = awaitItem()
            assertFalse(initial.isSuccess)

            viewModel.onAction(LoginUiAction.Submit)

            val loading = awaitItem()
            assertTrue(loading.isLoading)

            val success = awaitItem()
            assertTrue(success.isSuccess)
            assertFalse(success.isLoading)
        }
    }
}
```
