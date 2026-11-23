# 固件质量、测试与门禁规范 (TESTING)

规范编号：DEV-003  
版本：1.0.0  
适用范围：强制执行  

---

## 1. 质量保障体系全景

嵌入式固件由于运行于物理硬件，一旦发生 Crash 往往导致系统宕机甚至硬件变砖，因此必须遵循严格的多层次验证体系：

```
┌────────────────────────────────────────────────────────┐
│            1. 静态代码分析与格式门禁 (Static Checks)    │
│  - Clang-Format (Google/LLVM 规范，缩进 4 空格)        │
│  - Cppcheck (零内存泄漏、零空指针、零未初始化变量)       │
│  - GCC 编译告警控制 (-Wall -Wextra -Werror)            │
└──────────────────────────┬─────────────────────────────┘
                           ▼
┌────────────────────────────────────────────────────────┐
│            2. 算法与协议单元测试 (Unit Testing)         │
│  - Unity 测试框架覆盖 Gamma 查表、CCT 计算与 HSV 转换   │
│  - CMock 隔离硬件外设，验证协议大端序解包与 YBOT 验签  │
└──────────────────────────┬─────────────────────────────┘
                           ▼
┌────────────────────────────────────────────────────────┐
│            3. 编译构建与固件体积门禁 (Build Verification)│
│  - idf.py build: 0 错误 / 0 警告                       │
│  - 固件体积检查: 严格不得超出 ota 分区上限 (1536KB)    │
└────────────────────────────────────────────────────────┘
```

---

## 2. 自动化执行命令

AI 在完成代码编写后，必须依次运行以下命令验证：

```bash
# 1. 代码格式化检查
clang-format -i main/*.c main/*.h components/*/*/*.c components/*/*/*.h

# 2. 静态代码缺陷扫描
cppcheck --enable=all --inconclusive --error-exitcode=1 \
  --suppress=missingIncludeSystem \
  main/ components/

# 3. 固件编译构建验证
idf.py build

# 4. 运行单元测试 (Host 模拟或 Target 实机)
cd test && idf.py build && idf.py flash test
```

---

## 3. Unity 单元测试编写规范

所有纯数学算法（Gamma 查找表、CCT 色温插值、HSV 转换）、数据加密解密与协议解包必须配备 Unity 单元测试用例：

```c
#include "unity.h"
#include "lighting_cct.h"

void test_cct_power_conservation(void) {
    uint32_t duty_ww = 0, duty_cw = 0;
    
    // 测试 2700K 纯暖光
    esp_err_t err = lighting_cct_calculate(2700, 100, &duty_ww, &duty_cw);
    TEST_ASSERT_EQUAL(ESP_OK, err);
    TEST_ASSERT_EQUAL(8191, duty_ww);
    TEST_ASSERT_EQUAL(0, duty_cw);

    // 测试 4600K 中间色温功率恒定
    err = lighting_cct_calculate(4600, 100, &duty_ww, &duty_cw);
    TEST_ASSERT_EQUAL(ESP_OK, err);
    TEST_ASSERT_EQUAL(8191, duty_ww + duty_cw);
}

void test_null_pointer_safety(void) {
    esp_err_t err = lighting_cct_calculate(4000, 100, NULL, NULL);
    TEST_ASSERT_EQUAL(ESP_ERR_INVALID_ARG, err);
}
```

---

## 4. 固件体积与堆栈高水位监控

1. **固件体积门禁**：
   - 每次编译后必须检查 `build/*.bin` 体积，严禁超出 `partitions.csv` 中分配的 OTA 分区上限（默认 1.5MB），预留至少 15% 余量。
2. **任务堆栈高水位监控**：
   - 任务在开发调试期必须定期调用 `uxTaskGetStackHighWaterMark()` 检查历史最小剩余堆栈深度。
   - 若任何任务历史最小剩余堆栈小于 512 字节，必须增加任务分配堆栈，杜绝堆栈下溢踩踏系统内存。
