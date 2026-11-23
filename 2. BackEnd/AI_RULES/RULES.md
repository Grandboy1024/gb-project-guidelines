# AI 统一规范清单 (Rules Manifest)

版本：1.0.0  
适用项目：FastAPI 云端业务与后台服务 (Python BackEnd)  
最后更新：2026-11-23  

---

## 1. 规范定位与目的

本文档是本项目所有 AI 工具、智能助手（如 Cursor、Claude Code、GitHub Copilot、Gemini CLI、Antigravity、OpenDevin 等）的**唯一权威主入口**与规范索引清单。

所有 AI 在执行任何需求开发、缺陷修复、架构重构或测试编写任务前，**必须首先读取并严格遵守本清单及关联规范**。

本清单明确了各项规范的权威层级、任务适用矩阵以及冲突仲裁优先级。

---

## 2. 强制性核心规范 (Mandatory Core Rules)

以下文档对所有 AI 工具、所有开发场景均具有最高强制执行力：

| 规范编号 | 文件路径 | 核心内容概述 |
|---|---|---|
| **CORE-001** | [core/MASTER.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/2.%20BackEnd/AI_RULES/core/MASTER.md) | AI 角色定位、行为准则与 10 条绝对红线禁令 |
| **CORE-002** | [core/WORKFLOW.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/2.%20BackEnd/AI_RULES/core/WORKFLOW.md) | 5 阶段标准化作业流程 (SOP) 与任务执行规范 |
| **CORE-003** | [core/COMMUNICATION.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/2.%20BackEnd/AI_RULES/core/COMMUNICATION.md) | 沟通规范、代码变更呈现与决策对齐机制 |

---

## 3. 开发规范 (Development Rules)

在进行 Python / FastAPI 代码编写、接口定义、ORM 迁移或测试时，必须严格遵守以下规范：

| 规范编号 | 文件路径 | 核心内容概述 |
|---|---|---|
| **DEV-001** | [development/CODING.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/2.%20BackEnd/AI_RULES/development/CODING.md) | Python 3.12+ 语法、Pydantic V2、统一错误信封、SQL 事务与边界防御 |
| **DEV-002** | [development/ARCHITECTURE.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/2.%20BackEnd/AI_RULES/development/ARCHITECTURE.md) | 领域模块化单体、Router/Service/Repository/Model 分层、依赖注入、异步 Session |
| **DEV-003** | [development/TESTING.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/2.%20BackEnd/AI_RULES/development/TESTING.md) | pytest + pytest-asyncio 异步单测、httpx 接口集成测试、Alembic 迁移与 Ruff/Mypy 门禁 |

---

## 4. 项目知识库 (Project Knowledge)

以下文档提供项目的业务全景、历史技术决策与当前进展：

| 规范编号 | 文件路径 | 核心内容概述 |
|---|---|---|
| **PROJECT-001** | [project/CONTEXT.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/2.%20BackEnd/AI_RULES/project/CONTEXT.md) | 业务背景、产品定位与核心领域模块划分 |
| **PROJECT-002** | [project/DECISIONS.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/2.%20BackEnd/AI_RULES/project/DECISIONS.md) | 已确认的技术架构决策 (ADR-001 ~ ADR-008) |
| **PROJECT-003** | [project/STATUS.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/2.%20BackEnd/AI_RULES/project/STATUS.md) | 项目当前进展阶段、已完成模块与后续演进目标 |

---

## 5. 核心技术选型速查表

AI 生成的所有代码必须 100% 契合以下技术决策，严禁违背：

| 技术领域 | 选型方案 | 对应规范索引 |
|---|---|---|
| **开发语言 & 运行时** | Python 3.12+ 强类型注解 | `DEV-001` / `FastAPI 项目规范.md` 第一节 |
| **Web 框架** | `FastAPI` (基于 Starlette 与 ASGI) | `DEV-002` / `FastAPI 项目规范.md` 第一节 |
| **架构分层** | 领域模块化单体 (Domains: Router → Service → Repository → Model) | `DEV-002` / `FastAPI 项目规范.md` 第三节 |
| **数据校验与序列化** | `Pydantic V2` (`BaseModel`, `ConfigDict(from_attributes=True)`) | `DEV-001` / `FastAPI 项目规范.md` 第四节 |
| **ORM 与持久化** | `SQLAlchemy 2.0` (完全异步体系 `AsyncSession` + `selectinload`) | `DEV-002` / `FastAPI 项目规范.md` 第五节 |
| **数据库驱动** | `asyncpg` (PostgreSQL 17/18 官方异步连接池驱动) | `DEV-002` / `FastAPI 项目规范.md` 第五节 |
| **数据库版本迁移** | `Alembic` (所有变更生成迁移脚本，严禁直接手改 DB) | `DEV-002` / `FastAPI 项目规范.md` 第五节 |
| **缓存与临时状态** | `redis-py` (原生 `asyncio` 异步客户端) | `DEV-002` / `FastAPI 项目规范.md` 第六节 |
| **依赖注入** | `fastapi.Depends` (统一由 `deps.py` 暴露 `get_db`、`get_current_user` 等) | `DEV-002` / `FastAPI 项目规范.md` 第六节 |
| **环境与配置管理** | `pydantic-settings` (`BaseSettings` 集中配置，禁止裸用 `os.getenv`) | `DEV-001` / `FastAPI 项目规范.md` 第八节 |
| **日志与链路追踪** | `structlog` (统一 JSON 结构化日志，拦截并透传 `traceId`) | `DEV-001` / `FastAPI 项目规范.md` 第九节 |
| **异常与错误契约** | 自定义 `AppException` 密封类层级 + 全局 Exception Handler + 统一错误信封 | `DEV-001` / `FastAPI 项目规范.md` 第七节 |
| **测试框架** | `pytest` + `pytest-asyncio` + `httpx.AsyncClient` | `DEV-003` / `FastAPI 项目规范.md` 第十节 |
| **代码规范与包管理** | `uv` (包与虚拟环境管理) + `Ruff` (Linter & Formatter) + `Mypy` (静态检查) | `DEV-003` / `FastAPI 项目规范.md` 第十节 |

---

## 6. 执行准则与约束

1. **先读后写**：在编写或修改任何代码前，必须先查阅本清单并定位具体规范。
2. **严格守约**：不得因便利性擅自简化或绕过分层架构（如直接在 Router 中写 SQL 查询或业务逻辑）。
3. **禁止擅自重新定义**：未经开发者明确指令，严禁自行推翻既有技术选型或引入同步阻塞库（如 `requests`、同步 `psycopg2` 等）。
4. **数据库变更必出迁移**：凡涉及 SQLAlchemy Model 字段增删改，必须生成对应的 Alembic 迁移脚本并验证可逆性。
5. **及时报告缺失**：若发现引用的规范文件不存在，必须第一时间向开发者报告。

---

## 7. 文档冲突优先级

当不同指令或文档出现冲突时，严格按照以下优先级从高到低仲裁：

1. 系统与平台安全、运行环境与生产数据库保护约束。
2. 开发者针对当前任务给出的显式指令。
3. 已确认的架构决策记录 (`project/DECISIONS.md`)。
4. 核心行为准则与红线 (`core/MASTER.md`, `core/WORKFLOW.md`)。
5. 任务相关的开发规范 (`development/CODING.md`, `ARCHITECTURE.md`, `TESTING.md`)。
6. 项目背景与状态记录 (`project/CONTEXT.md`, `STATUS.md`)。

---

## 8. 规范维护机制

本文档是项目规范的最高登记处。
当新增、移除或调整任何规范文档时，必须同步更新本清单并在 [CHANGELOG.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/2.%20BackEnd/AI_RULES/CHANGELOG.md) 中记录版本演进。
