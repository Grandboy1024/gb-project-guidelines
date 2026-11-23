# 规范演进与变更记录 (CHANGELOG)

本文档用于追踪并记录本 FastAPI 后端服务项目 AI 规范体系与架构约定的所有版本变更。

---

## [1.0.0] - 2026-11-23

### 初始发布 (Initial Release)
- **体系建立**：全面落地三层 AI 协作规范架构（通用规范层 `AI_RULES/` + 平台适配层 + 项目知识库）。
- **总清单发布**：发布 [RULES.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/2.%20BackEnd/AI_RULES/RULES.md)，确立权威文件索引、执行准则与文档冲突优先级。
- **核心行为准则**：
  - [core/MASTER.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/2.%20BackEnd/AI_RULES/core/MASTER.md)：定义资深后端架构师角色定位、四项基本行为准则与十条绝对禁止的硬性红线。
  - [core/WORKFLOW.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/2.%20BackEnd/AI_RULES/core/WORKFLOW.md)：确立标准五步法作业流程 (SOP)。
  - [core/COMMUNICATION.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/2.%20BackEnd/AI_RULES/core/COMMUNICATION.md)：规范汇报模版、可点击文件链接与冲突处理原则。
- **开发与设计规范**：
  - [development/CODING.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/2.%20BackEnd/AI_RULES/development/CODING.md)：规范 Python 3.12+ 强类型注解、Pydantic V2 数据校验、统一响应体契约、SQLAlchemy 2.0 异步 Session 与防 SQL 注入/防并发竞争。
  - [development/ARCHITECTURE.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/2.%20BackEnd/AI_RULES/development/ARCHITECTURE.md)：确立 Domain 领域模块化单体、Router -> Service -> Repository -> Model 依赖单向流动、FastAPI 依赖注入与全局生命周期管理。
  - [development/TESTING.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/2.%20BackEnd/AI_RULES/development/TESTING.md)：规范 pytest + pytest-asyncio 异步测试、httpx.AsyncClient 集成测试、数据库迁移回滚测试与 Ruff/Mypy 门禁。
- **项目知识资产**：
  - [project/CONTEXT.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/2.%20BackEnd/AI_RULES/project/CONTEXT.md)：明确智能灯云平台与后台服务的业务全景与领域划分。
  - [project/DECISIONS.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/2.%20BackEnd/AI_RULES/project/DECISIONS.md)：固化架构决策记录 ADR-001 至 ADR-008。
  - [project/STATUS.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/2.%20BackEnd/AI_RULES/project/STATUS.md)：记录当前架构基线就绪状态与后续演进目标。
- **适配层入口**：
  - 发布适配 Codex / Copilot 的 [AGENTS.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/2.%20BackEnd/AGENTS.md)。
  - 发布适配 Claude Code 的 [CLAUDE.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/2.%20BackEnd/CLAUDE.md)。
  - 发布适配 Gemini CLI / Antigravity 的 [GEMINI.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/2.%20BackEnd/GEMINI.md)。
