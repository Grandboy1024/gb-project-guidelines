# 规范演进与变更记录 (CHANGELOG)

本文档用于追踪并记录本 React Web 管理后台项目 AI 规范体系与架构约定的所有版本变更。

---

## [1.0.0] - 2026-11-23

### 初始发布 (Initial Release)
- **体系建立**：全面落地三层 AI 协作规范架构（通用规范层 `AI_RULES/` + 平台适配层 + 项目知识库）。
- **总清单发布**：发布 [RULES.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/4.%20FrontEnd/AI_RULES/RULES.md)，确立权威文件索引、执行准则与文档冲突优先级。
- **核心行为准则**：
  - [core/MASTER.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/4.%20FrontEnd/AI_RULES/core/MASTER.md)：定义资深前端架构师角色定位、四项基本行为准则与十条绝对禁止的硬性红线。
  - [core/WORKFLOW.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/4.%20FrontEnd/AI_RULES/core/WORKFLOW.md)：确立标准五步法作业流程 (SOP)。
  - [core/COMMUNICATION.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/4.%20FrontEnd/AI_RULES/core/COMMUNICATION.md)：规范汇报模版、可点击文件链接与冲突处理原则。
- **开发与设计规范**：
  - [development/CODING.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/4.%20FrontEnd/AI_RULES/development/CODING.md)：规范 TypeScript 严格模式、Zod 运行时校验、UI 四态闭环、长文本防溢出、按钮防重复点击与输入防抖。
  - [development/ARCHITECTURE.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/4.%20FrontEnd/AI_RULES/development/ARCHITECTURE.md)：确立 Domain 领域就近聚合 (Colocation)、TanStack Query 服务端缓存与 Zustand 客户端轻量状态彻底分离、统一网络层拦截与数据解包。
  - [development/TESTING.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/4.%20FrontEnd/AI_RULES/development/TESTING.md)：规范 Vitest + React Testing Library 单元/Hook 测试、MSW 接口 Mock 与质量门禁。
- **项目知识资产**：
  - [project/CONTEXT.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/4.%20FrontEnd/AI_RULES/project/CONTEXT.md)：明确智能灯云平台 Web 管理后台的产品定位与领域划分。
  - [project/DECISIONS.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/4.%20FrontEnd/AI_RULES/project/DECISIONS.md)：固化架构决策记录 ADR-001 至 ADR-008。
  - [project/STATUS.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/4.%20FrontEnd/AI_RULES/project/STATUS.md)：记录当前架构基线就绪状态与后续演进目标。
- **适配层入口**：
  - 发布适配 Codex / Copilot 的 [AGENTS.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/4.%20FrontEnd/AGENTS.md)。
  - 发布适配 Claude Code 的 [CLAUDE.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/4.%20FrontEnd/CLAUDE.md)。
  - 发布适配 Gemini CLI / Antigravity 的 [GEMINI.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/4.%20FrontEnd/GEMINI.md)。
