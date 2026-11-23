# 架构决策记录 (DECISIONS)

规范编号：PROJECT-002  
版本：1.0.0  
适用范围：强制执行  

---

## ADR-001: 采用 FastAPI 作为唯一核心后端 Web 框架

- **决策状态**：已确认 (Confirmed)
- **决策背景**：系统需要同时面向高并发移动端请求与物联网长连接/高频状态同步，传统同步框架（如 Django/Flask）在 I/O 密集场景下线程开销大、并发瓶颈明显。
- **决策结论**：全工程统一采用 **FastAPI (ASGI)** 作为核心 Web 框架。
- **强制规则**：
  - 核心接口必须全异步定义 (`async def`)。
  - 充分利用原生依赖注入系统 (`fastapi.Depends`) 进行解耦与测试 Mock。

---

## ADR-002: 采用领域模块化单体 (Modular Monolith) 架构

- **决策状态**：已确认 (Confirmed)
- **决策背景**：项目初期预计 1.5 万用户，过早拆分微服务会引入网络延迟、分布式事务、跨服务部署及复杂排障成本。
- **决策结论**：采用 **领域模块化单体架构**，按 Domain（`auth`、`users`、`products`、`orders`、`devices`、`ota`）垂直划分。
- **强制规则**：
  - 领域内部包含 `models`、`schemas`、`repository`、`service`、`router`、`deps`、`exceptions`。
  - 跨域协同必须通过目标领域的公开 Service 或公开依赖，严禁私自跨域调用底层 Model 或 Repository。

---

## ADR-003: 采用 SQLAlchemy 2.0 异步体系与 PostgreSQL 17/18

- **决策状态**：已确认 (Confirmed)
- **决策背景**：业务数据具有高关系型结构要求；PostgreSQL 对 JSONB 扩展（设备物模型属性）与事务支持极其完善。
- **决策结论**：主数据库统一采用 **PostgreSQL**，ORM 统一采用 **SQLAlchemy 2.0 + asyncpg**。
- **强制规则**：
  - 会话统一使用 `AsyncSession`。
  - 严禁拼接裸 SQL 字符串；关联查询显式指定 `selectinload` / `joinedload`，坚决杜绝 N+1 查询。

---

## ADR-004: 采用 Alembic 进行版本化双向可逆数据库迁移

- **决策状态**：已确认 (Confirmed)
- **决策背景**：手动执行 DDL 极易导致各环境数据库结构不一致，甚至导致生产环境灾难性数据损坏。
- **决策结论**：所有数据库表结构与索引变更必须通过 **Alembic 迁移脚本** 落地。
- **强制规则**：
  - 严禁在生产或代码初始化中调用 `Base.metadata.create_all()`。
  - 每个迁移脚本必须完整实现 `upgrade()` 与 `downgrade()`，确保可回滚。

---

## ADR-005: 采用 Pydantic V2 契约与统一错误信封

- **决策状态**：已确认 (Confirmed)
- **决策背景**：前后端分离开发中，随意的入参及报错结构会导致客户端频繁解析崩溃。
- **决策结论**：统一采用 **Pydantic V2** 进行请求校验与响应序列化；统一输出标准错误响应信封（遵循《统一错误码与异常处理规范》）。
- **强制规则**：
  - 对外 API 字段一律转换为 `camelCase` 驼峰契约。
  - 未捕获异常一律转换为统一错误信封，透传分布式追踪 `traceId`，禁止裸漏 500 堆栈。

---

## ADR-006: 采用 pydantic-settings 集中管理环境配置并默认兜底 prod

- **决策状态**：已确认 (Confirmed)
- **决策背景**：散落的 `os.getenv` 容易出现配置缺漏或测试密钥泄露到生产环境。
- **决策结论**：配置统一由 `app/core/config.py` 中的 `Settings(BaseSettings)` 接管。
- **强制规则**：
  - 无参数默认运行永远指向生产 `prod` 约束，必须显式通过 `.env` 或环境变量注入开发/测试连接。
  - 敏感密钥严禁提交至 Git 仓库。

---

## ADR-007: 采用 Structlog 与全链路 TraceId 追踪

- **决策状态**：已确认 (Confirmed)
- **决策背景**：传统的文本日志格式混乱，难以被 ELK / CloudWatch 结构化检索与统计。
- **决策结论**：日志统一采用 **Structlog 输出 JSON 格式**，中间件拦截并透传 `traceId`。
- **强制规则**：
  - 严禁在任何业务代码中使用 `print()`。
  - 每条日志必须附带 `traceId`、时间戳与事件上下文。

---

## ADR-008: 采用 uv + Ruff + Mypy 作为现代 Python 工程化基石

- **决策状态**：已确认 (Confirmed)
- **决策背景**：传统的 pip / poetry 在依赖解析速度与虚拟环境管理上效率低下；flake8 / black / isort 多工具混用导致维护复杂。
- **决策结论**：统一采用 **uv** 进行极速包管理与确定性锁定（`uv.lock`）；采用 **Ruff** 进行极速格式化与代码质量校验；采用 **Mypy** 进行强静态类型检查。
- **强制规则**：
  - 本地与 CI 必须通过 `ruff check`、`ruff format --check` 与 `mypy`。
