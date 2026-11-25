# AI 沟通与交互规范 (COMMUNICATION)

规范编号：CORE-003  
版本：1.0.0  
适用范围：强制执行  

---

## 1. 核心沟通原则

1. **精准精炼、结果导向**：
   - 杜绝多余的寒暄与废话，以条理清晰的技术语言直接回应。
   - 采用标准 GitHub Flavored Markdown 组织内容。
2. **所有文件必须提供可点击的 Markdown 链接**：
   - 凡是在回复中提及的文件，必须使用标准 `file://` 格式的 Markdown 链接，如 `[driver_ledc.c](file:///absolute/path/to/driver_ledc.c#L10-L25)`。
   - 链接文本统一使用文件基础名称（Basename），保持排版易读。
3. **硬件资源变动透明化**：
   - 涉及 GPIO 引脚、硬件定时器、ADC 通道或 FreeRTOS 任务分配时，必须以表格形式清晰展示资源占用情况，严禁悄无声息地改变引脚绑定。
4. **决策与权衡重点说明**：
   - 涉及重要技术选型或架构取舍（如为何选择预分配环形缓冲区而非动态 `malloc`、为何选择 LEDC 硬件渐变而非软件延时调光），简要说明设计理由与可靠性考量。

---

## 2. 任务交付标准汇报模版

当完成功能开发、驱动适配或 Bug 修复任务后，按照以下格式向开发者交付：

```markdown
### 1. 任务完成概括
- 简要总结完成了哪些固件组件、驱动模块、通信协议或业务链路的开发与优化。

### 2. 变更文件清单
- [bsp_pins.h](file:///path/to/components/bsp/include/bsp_pins.h) - 新增硬件引脚映射与外设通道常量。
- [driver_ledc.c](file:///path/to/components/drivers/src/driver_ledc.c) - 20kHz 高频 PWM 驱动与硬件渐变实现。
- [lighting_cct.c](file:///path/to/components/lighting_engine/src/lighting_cct.c) - 2700K~6500K 双色温功率恒定插值算法。
- [ble_gatts.c](file:///path/to/components/protocols/ble/src/ble_gatts.c) - 对齐《BLE API 规范》大端序指令解析与 `0xFF55`/厂商数据广播。
- [ota_upgrade.c](file:///path/to/components/ota_manager/src/ota_upgrade.c) - A/B 双分区写入、YBOT 头部验签与防回滚自检。
- [test_lighting.c](file:///path/to/test/test_lighting.c) - Unity 单元测试用例。

### 3. 硬件资源与任务开销评估
| 资源类别 | 分配详情 | 说明 |
|---|---|---|
| **GPIO 引脚** | GPIO 4 (PWM CW), GPIO 5 (PWM WW), GPIO 6 (PIR 中断) | 无引脚冲突 |
| **硬件通道** | LEDC_TIMER_0, LEDC_CHANNEL_0 / 1 (20kHz, 13-bit) | 高频无频闪 |
| **FreeRTOS 任务** | `ctrl_task` (优先级 5, 栈 4096B), `sensor_task` (优先级 3, 栈 3072B) | 高水位余量 > 1KB |

### 4. 工具链与质量门禁验证
- `clang-format`: 代码格式化完毕，0 样式违规。
- `cppcheck`: 静态检查通过，0 内存泄漏 / 0 空指针警告。
- `idf.py build`: 固件编译通过，0 错误 / 0 警告，生成产物大小为 842KB (分区上限 1536KB)。
- `Unity Test`: 关键算法与协议解析用例全部绿灯通过。

### 5. 关键防御性与可靠性实现说明
- 简要列出针对该任务实施的防御性设计（如空指针入参校验、LEDC 硬件定时器防频闪、NVS 2秒防抖防磨损、Task Watchdog 注册喂狗、OTA 签名校验与双分区自动回滚等）。
```

---

## 3. 歧义与冲突处理机制

1. **架构冲突纠正**：
   - 若开发者的指令与项目核心固件规范相违背（例如要求“在 ISR 中直接用 printf 打印调试日志”或“在主循环中不断 malloc 申请内存”），**绝对不能盲从**。
   - 必须以礼貌专业的态度指出该做法违背了规范 (`CORE-001`, `DEV-001`)，并主动给出符合实时系统安全标准的替代方案（如使用 `xQueueSendFromISR` 将事件交由工作任务异步处理）。
2. **硬件引脚与外设冲突**：
   - 若发现任务要求的 GPIO 引脚已被 Strapping 引脚、JTAG 调试引脚或已存在的 SPI/I2C 总线占用，不得随意选择未验证的引脚，必须主动列出冲突清单并请求开发者裁决。
3. **破坏性与高危操作**：
   - 涉及整片 Flash 擦除 (`esptool.py erase_flash`)、修改 Bootloader 核心代码或变更出厂配网加密凭据时，必须显式提醒风险并确认。
