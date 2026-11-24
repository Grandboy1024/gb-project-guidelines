# 规范演进与变更记录 (CHANGELOG)

本文档用于追踪并记录本 Flutter 移动应用项目 AI 规范体系与架构约定的所有版本变更。

---

## [1.0.0] - 2026-11-20

### 初始发布 (Initial Release)
- **体系建立**：全面落地三层 AI 协作规范架构（通用规范层 `AI_RULES/` + 平台适配层 + 项目知识库）。
- **总清单发布**：发布 [RULES.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/3.%20App/AI_RULES/RULES.md)，确立权威文件索引、执行准则与文档冲突优先级。
- **核心行为准则**：
  - [core/MASTER.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/3.%20App/AI_RULES/core/MASTER.md)：定义架构师角色定位、四项基本行为准则与十条绝对禁止的硬性红线。
  - [core/WORKFLOW.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/3.%20App/AI_RULES/core/WORKFLOW.md)：确立标准五步法作业流程 (SOP)。
  - [core/COMMUNICATION.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/3.%20App/AI_RULES/core/COMMUNICATION.md)：规范汇报模版、可点击文件链接与冲突处理原则。
- **开发与设计规范**：
  - [development/CODING.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/3.%20App/AI_RULES/development/CODING.md)：规范 Dart 3 标准、页面就近聚合、Design Tokens（`AppSpacing`、`AppRadius`、`Gap`）以及零崩溃防御性 UI。
  - [development/ARCHITECTURE.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/3.%20App/AI_RULES/development/ARCHITECTURE.md)：确立 Clean MVVM 单向数据流、Riverpod 2.x 代码生成、Typed GoRouter 强类型路由、请求对象模式与按 Feature 隔离的国际化方案。
  - [development/TESTING.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/3.%20App/AI_RULES/development/TESTING.md)：规范 `ProviderContainer` 单元测试、Mocktail 仓储测试与 L10n 一致性测试套件。
- **项目知识资产**：
  - [project/CONTEXT.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/3.%20App/AI_RULES/project/CONTEXT.md)：明确移动端产品定位与 Feature 模块全景。
  - [project/DECISIONS.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/3.%20App/AI_RULES/project/DECISIONS.md)：固化架构决策记录 ADR-001 至 ADR-008。
  - [project/STATUS.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/3.%20App/AI_RULES/project/STATUS.md)：记录当前架构基线就绪状态与后续演进目标。
- **适配层入口**：
  - 发布适配 Codex / Copilot 的 [AGENTS.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/3.%20App/AGENTS.md)。
  - 发布适配 Claude Code 的 [CLAUDE.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/3.%20App/CLAUDE.md)。
  - 发布适配 Gemini CLI / Antigravity 的 [GEMINI.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/3.%20App/GEMINI.md)。
- **全量汉化**：全套规范文档及平台入口完全转换为规范、专业的中文技术文档。
