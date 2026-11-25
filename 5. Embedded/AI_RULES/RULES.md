# AI 统一规范清单 (Rules Manifest)

版本：1.0.0  
适用项目：ESP-IDF 嵌入式固件与 IoT 硬件设备 (C / FreeRTOS Embedded)  
最后更新：2026-11-23  

---

## 1. 规范定位与目的

本文档是本项目所有 AI 工具、智能助手（如 Cursor、Claude Code、GitHub Copilot、Gemini CLI、Antigravity、OpenDevin 等）的**唯一权威主入口**与规范索引清单。

所有 AI 在执行任何固件需求开发、驱动适配、协议联调、缺陷修复或架构重构前，**必须首先读取并严格遵守本清单及关联规范**。

本清单明确了各项规范的权威层级、任务适用矩阵以及冲突仲裁优先级。

---

## 2. 强制性核心规范 (Mandatory Core Rules)

以下文档对所有 AI 工具、所有开发场景均具有最高强制执行力：

| 规范编号 | 文件路径 | 核心内容概述 |
|---|---|---|
| **CORE-001** | [core/MASTER.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/5.%20Embedded/AI_RULES/core/MASTER.md) | AI 角色定位、行为准则与 10 条绝对红线禁令 |
| **CORE-002** | [core/WORKFLOW.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/5.%20Embedded/AI_RULES/core/WORKFLOW.md) | 5 阶段标准化作业流程 (SOP) 与任务执行规范 |
| **CORE-003** | [core/COMMUNICATION.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/5.%20Embedded/AI_RULES/core/COMMUNICATION.md) | 沟通规范、代码变更呈现与硬件决策对齐机制 |

---

## 3. 开发规范 (Development Rules)

在进行 C / FreeRTOS 固件代码编写、外设驱动适配、通信协议实现或测试时，必须严格遵守以下规范：

| 规范编号 | 文件路径 | 核心内容概述 |
|---|---|---|
| **DEV-001** | [development/CODING.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/5.%20Embedded/AI_RULES/development/CODING.md) | C99/C11 严格规范、内存静态化、ISR 极速准则、Gamma 2.2 调光、NVS 磨损防抖与指针防御 |
| **DEV-002** | [development/ARCHITECTURE.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/5.%20Embedded/AI_RULES/development/ARCHITECTURE.md) | ESP-IDF 组件化垂直分层、FreeRTOS 多任务与优先级规划、线程安全 IPC 队列、A/B 双分区 OTA 机制 |
| **DEV-003** | [development/TESTING.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/5.%20Embedded/AI_RULES/development/TESTING.md) | Unity 单元测试框架、CMock 硬件抽象 Mock、Cppcheck / Clang-Format 门禁与编译检查 |

---

## 4. 项目知识库 (Project Knowledge)

以下文档提供项目的业务全景、历史技术决策与当前进展：

| 规范编号 | 文件路径 | 核心内容概述 |
|---|---|---|
| **PROJECT-001** | [project/CONTEXT.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/5.%20Embedded/AI_RULES/project/CONTEXT.md) | 业务背景、产品定位与核心硬件组件划分 |
| **PROJECT-002** | [project/DECISIONS.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/5.%20Embedded/AI_RULES/project/DECISIONS.md) | 已确认的固件架构决策 (ADR-001 ~ ADR-008) |
| **PROJECT-003** | [project/STATUS.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/5.%20Embedded/AI_RULES/project/STATUS.md) | 项目当前进展阶段、已完成组件与后续演进目标 |

---

## 5. 核心技术选型速查表

AI 生成的所有代码必须 100% 契合以下技术决策，严禁违背：

| 技术领域 | 选型方案 | 对应规范索引 |
|---|---|---|
| **主控芯片 & 硬件平台** | `ESP32-C3` / `ESP32-S3`（32-bit RISC-V/Xtensa，支持 Wi-Fi + BLE 5.0，备选 STM32G0/L4） | `PROJECT-001` / `ESP-IDF 固件项目规范.md` 第一节 |
| **操作系统与运行时** | `FreeRTOS` (ESP-IDF v5.x 原生多任务抢占式内核) | `DEV-002` / `ESP-IDF 固件项目规范.md` 第一节 |
| **固件框架与架构** | `ESP-IDF Component-based Architecture` (组件化高内聚低耦合分层) | `DEV-002` / `ESP-IDF 固件项目规范.md` 第二节 |
| **编程语言标准** | `C99 / C11` 严格规范（局部 C++17 受限子集，禁用异常、RTTI 与动态内存滥用） | `DEV-001` / `ESP-IDF 固件项目规范.md` 第一节 |
| **蓝牙通信协议栈** | `NimBLE / Bluedroid` (BLE 5.0 GATT Server，对齐《BLE API 规范》31B 主广播含 `0xFF55` UUID、`0xFF` 厂商数据与 `DeviceID_Short`) | `DEV-001` / `ESP-IDF 固件项目规范.md` 第四节 |
| **物联网网络协议** | `esp-mqtt` (MQTT 3.1.1/5.0 over TLS 1.2/1.3) + `esp_wifi` 配网，严格对齐《MQTT 规范》 | `DEV-002` / `ESP-IDF 固件项目规范.md` 第四节 |
| **空中升级与防变砖** | `A/B 双分区 OTA` + `Bootloader` 自动健康自检与失败回滚，对齐《OTA 固件升级与设备管理规范》 | `DEV-002` / `ESP-IDF 固件项目规范.md` 第六节 |
| **数字签名与固件安全** | `YBOT` 二进制信封格式 + `ECDSA-SHA256` 离线私钥验签 + `Anti-Rollback` 防降级计数器 | `DEV-002` / `ESP-IDF 固件项目规范.md` 第六节 |
| **持久化存储** | `NVS (Non-Volatile Storage)`，Flash 磨损均衡机制，状态变更 2 秒防抖落盘 | `DEV-001` / `ESP-IDF 固件项目规范.md` 第七节 |
| **调光算法与 PWM** | `LEDC 硬件定时器 PWM` (频率 $\ge$ 20kHz 无频闪，13-bit 分辨率，Gamma 2.2 查找表，CCT 色温插值) | `DEV-001` / `ESP-IDF 固件项目规范.md` 第五节 |
| **模拟与外设驱动** | `ADC Calibration` (移动均值滤波，锂电池放电曲线映射，欠压分级降额) + `GPIO 中断防抖` | `DEV-001` / `ESP-IDF 固件项目规范.md` 第五节 |
| **低功耗电源管理** | `ESP-IDF Power Management` (Light Sleep / Deep Sleep，DFS 动态降频，待机功耗 < 0.5W) | `DEV-002` / `ESP-IDF 固件项目规范.md` 第八节 |
| **系统容错与看门狗** | `Task Watchdog Timer (TWDT)` + 硬件看门狗 + `Flash Core Dump` 崩溃上下文持久化 | `DEV-001` / `ESP-IDF 固件项目规范.md` 第七节 |
| **结构化日志** | `ESP_LOGx` 统一分级日志（禁止生产代码裸用 `printf`，生产环境日志收敛至 `INFO`） | `DEV-001` / `ESP-IDF 固件项目规范.md` 第九节 |
| **构建与质量门禁** | `CMake` + `Ninja` + `idf.py`；`Clang-Format` + `Cppcheck` + `Unity / CMock` 自动化测试 | `DEV-003` / `ESP-IDF 固件项目规范.md` 第十节 |

---

## 6. 执行准则与约束

1. **先查后写**：编写或修改任何外设驱动与通信逻辑前，必须查阅相关硬件引脚分配表 (Pinout) 与既有组件代码。
2. **严防内存碎片**：严禁在主业务循环或 ISR 中调用 `malloc()` / `free()`，优先采用静态分配或全局对象池。
3. **严格中断守则**：中断服务例程 (ISR) 严禁执行任何阻塞 API、浮点计算、日志打印或动态内存分配，必须做到极速退出。
4. **防变砖铁律**：OTA 必须执行硬件兼容性比对、ECDSA 数字签名验证、CRC32 校验与电量检查 ($\ge 30\%$)，并具备双分区自动回滚能力。
5. **严密看门狗防护**：主业务任务与通信任务必须注册 Task Watchdog Timer，并在主循环中按时喂狗，杜绝死锁与任务悬挂。

---

## 7. 文档冲突优先级

当不同指令或文档出现冲突时，严格按照以下优先级从高到低仲裁：

1. 硬件物理安全、过充/过温电气保护与防变砖安全机制。
2. 开发者针对当前任务给出的显式指令。
3. 已确认的架构决策记录 (`project/DECISIONS.md`)。
4. 核心行为准则与红线 (`core/MASTER.md`, `core/WORKFLOW.md`)。
5. 任务相关的开发规范 (`development/CODING.md`, `ARCHITECTURE.md`, `TESTING.md`)。
6. 项目背景与状态记录 (`project/CONTEXT.md`, `STATUS.md`)。

---

## 8. 规范维护机制

本文档是项目规范的最高登记处。
当新增、移除或调整任何规范文档时，必须同步更新本清单并在 [CHANGELOG.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/5.%20Embedded/AI_RULES/CHANGELOG.md) 中记录版本演进。
