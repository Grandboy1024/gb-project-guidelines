# 规范演进与变更记录 (CHANGELOG)

本文档用于追踪并记录本嵌入式固件与 IoT 硬件设备项目 AI 规范体系与架构约定的所有版本变更。

---

## [1.2.0] - 2026-11-25

### BLE API V1.1 协议基线对齐
- **帧协议与广播升级**：同步《BLE API 规范》V1.1 修订，广播 `AdvVer` 升级为 `0x02`，扩展 `CAP`（Bit 0~6）与 `STATE`（Bit 0~5）；应用帧升级为 10 字节固定开销（2 字节 `SEQ` Big-Endian，最大载荷 502B），支持连接级事务缓存去重、独立 `Operation` 操作语义、能力模型 `0x02`、RGBIC 分段、Wi-Fi 配网与 `0xC0`–`0xC9` OTA 子协议。
- **跨文档同步**：同步更新 `ESP-IDF 固件项目规范.md`（§4.1/§4.2）、`OTA 固件升级与设备管理规范.md`（§1.1/§3.1）与 `DECISIONS.md`（ADR-003）。

---

## [1.1.0] - 2026-11-25

### BLE 广播与跨文档契约对齐
- **广播结构升级**：同步《BLE API 规范》第二章修订，将 31 字节主广播统一为 `3B Flags + 4B Service UUID (0xFF55) + 24B 厂商数据 (0xFF)`，新增 `AdvVer`、`CAP`、`StateSeq` 与 `STATE.Bit2` 可配网窗口位（Control Service 采用 `0xFF55` 以避开他项目占用的 `0xFF77`）。
- **标识与安全修正**：废弃物理 MAC 异或 `0xAD` 混淆方案，统一采用 `DeviceID[2..7]` 填充 `DeviceID_Short`；将 GATT Write/Notify 特征升级为自定义 128-bit UUID（`A83B0002-...` / `A83B0003-...`），并同步更新 `CONTEXT.md`、`ESP-IDF 固件项目规范.md` 及 `DECISIONS.md`（ADR-003）。

---

## [1.0.0] - 2026-11-23

### 初始发布 (Initial Release)
- **体系建立**：全面落地三层 AI 协作规范架构（通用规范层 `AI_RULES/` + 平台适配层 + 项目知识库）。
- **总清单发布**：发布 [RULES.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/5.%20Embedded/AI_RULES/RULES.md)，确立权威文件索引、执行准则与文档冲突优先级。
- **核心行为准则**：
  - [core/MASTER.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/5.%20Embedded/AI_RULES/core/MASTER.md)：定义资深物联网固件与嵌入式系统架构师角色定位、四项基本行为准则与十条绝对禁止的硬性红线。
  - [core/WORKFLOW.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/5.%20Embedded/AI_RULES/core/WORKFLOW.md)：确立标准五步法作业流程 (SOP)。
  - [core/COMMUNICATION.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/5.%20Embedded/AI_RULES/core/COMMUNICATION.md)：规范交付汇报模版、可点击文件链接与硬件引脚/协议冲突对齐机制。
- **开发与设计规范**：
  - [development/CODING.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/5.%20Embedded/AI_RULES/development/CODING.md)：规范 C99/C11 严格编码、静态内存分配、ISR 极速退出、Gamma 2.2 调光曲线、NVS 磨损防抖与指针防御。
  - [development/ARCHITECTURE.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/5.%20Embedded/AI_RULES/development/ARCHITECTURE.md)：确立 ESP-IDF 组件化垂直分层、FreeRTOS 多任务调度与优先级规划、线程安全 IPC 队列通信、A/B 双分区防变砖 OTA 容灾机制。
  - [development/TESTING.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/5.%20Embedded/AI_RULES/development/TESTING.md)：规范 Unity 单元测试框架、CMock 硬件抽象隔离、Cppcheck / Clang-Format 代码质量门禁。
- **项目知识资产**：
  - [project/CONTEXT.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/5.%20Embedded/AI_RULES/project/CONTEXT.md)：明确智能照明、太阳能户外灯与智能音频硬件的产品定位与组件划分。
  - [project/DECISIONS.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/5.%20Embedded/AI_RULES/project/DECISIONS.md)：固化架构决策记录 ADR-001 至 ADR-008。
  - [project/STATUS.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/5.%20Embedded/AI_RULES/project/STATUS.md)：记录当前固件架构基线就绪状态与后续演进目标。
- **适配层入口**：
  - 发布适配 Codex / Copilot 的 [AGENTS.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/5.%20Embedded/AGENTS.md)。
  - 发布适配 Claude Code 的 [CLAUDE.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/5.%20Embedded/CLAUDE.md)。
  - 发布适配 Gemini CLI / Antigravity 的 [GEMINI.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/5.%20Embedded/GEMINI.md)。
