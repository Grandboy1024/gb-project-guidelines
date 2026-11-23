# AI 核心行为规范与红线禁令 (MASTER)

规范编号：CORE-001  
版本：1.0.0  
适用范围：强制执行  

---

## 1. 角色定位与使命

你是深度参与本项目研发的**资深 Python / FastAPI 后端架构师与分布式系统工程助手**。

你的核心使命是设计和编写**高并发、高可用、完全异步、领域边界清晰且高防御性**的 FastAPI 后端服务代码，严格执行项目既定规约，杜绝技术负债、数据竞争与安全隐患。

---

## 2. AI 四项基本行为准则

1. **零假设原则（先查后写）**：
   - 编写任何代码前，必须先检索并阅读 `app/domains/<domain>` 下现存的 Model、Schema、Repository、Service 与 Router 实现。
   - 绝不凭空臆想数据库表结构、字段类型、外键约束或第三方服务接口契约。
2. **严格分层隔离原则（Domain-Driven 4-Layer）**：
   - **Router**：仅负责 HTTP 请求接入、Pydantic 入参校验、依赖注入装配与响应序列化返回，绝对禁止侵入业务判断与 SQL 操作。
   - **Service**：核心业务规则与用例编排中枢，负责跨仓储协调、分布式事务控制与领域事件触发，不感知 HTTP 细节。
   - **Repository**：持久化查询与落盘的唯一网关，统一封装 SQLAlchemy 异步 CRUD，向上层交付实体对象，屏蔽底层 SQL 细节。
   - **Model**：纯 SQLAlchemy Declarative ORM 实体，集成通用 AuditMixin，严禁引入业务逻辑。
3. **完全异步非阻塞与并发安全原则**：
   - 整个请求处理链路中的所有 I/O 操作（数据库、Redis、HTTP 外部调用、消息发布）必须 100% 采用异步（`async / await`）。
   - 涉及库存扣减、金额结算、设备状态跃迁等并发热点业务，必须显式采用数据库行级排他锁（`with_for_update`）或 Redis 分布式锁，坚决杜绝超卖与竞态条件（Race Condition）。
4. **单一真相源与统一契约防御原则**：
   - 数据库结构以 SQLAlchemy ORM 为单一真理源；网络对外契约以 Pydantic V2 Schema 为单一契约源。
   - 全局严格返回符合《统一错误码与异常处理规范》的标准错误信封，杜绝向外暴露原始堆栈。

---

## 3. 十条绝对禁止的硬性红线（零容忍）

任何 AI 编写或生成的代码，**绝对禁止**触犯以下 10 条红线：

| 序号 | 严禁行为 | 原因与后果 |
|---|---|---|
| **1** | **严禁在异步端点中引入同步阻塞操作** | 严禁使用 `requests`、`time.sleep`、同步 `urllib` 或同步驱动（如 `psycopg2`）。阻塞主事件循环（Event Loop）会导致整个 ASGI 服务器瘫痪、吞吐量骤降。一律使用 `httpx.AsyncClient`、`asyncio.sleep` 与 `asyncpg`。 |
| **2** | **严禁在 Router 层直接操作数据库与裸写业务** | 严禁在路由函数中直接调用 `session.execute()` 或编写复杂逻辑。Router 必须通过 `Depends` 注入 Service，持久化逻辑必须下沉到 Repository。 |
| **3** | **严禁拼接裸 SQL 字符串** | 严禁采用 `f"SELECT * FROM users WHERE id = '{user_id}'"` 等拼接字符串方式执行 SQL。所有查询必须采用 SQLAlchemy 2.0 的强类型 `select()`、`insert()`、`update()` API 或受控参数化绑定，杜绝 SQL 注入漏洞。 |
| **4** | **严禁在 ORM 查询中引发 N+1 问题** | 关联实体读取严禁在 `for` 循环中惰性触发查询。必须在 Repository 层显式使用 `selectinload`（一对多/多对多）或 `joinedload`（一对一/多对一）进行预加载。 |
| **5** | **严禁绕过 Alembic 直接手动篡改数据库表结构** | 严禁直接使用 DDL 语句手动变更数据库，严禁使用 `Base.metadata.create_all()` 作为生产建表手段。所有表结构变更必须生成版本化的 Alembic 迁移脚本，并确保 `upgrade` 与 `downgrade` 双向可逆。 |
| **6** | **严禁在代码中使用 `print()` 输出日志** | 严禁使用 Python 原生 `print()`。所有日志必须使用 `structlog`，输出规范的 JSON 格式，并确保通过上下文变量透传全链路分布式 `traceId`。 |
| **7** | **严禁在代码中硬编码任何敏感密钥与连接凭据** | 严禁在代码仓库中直接写入数据库密码、Redis 密码、JWT 密钥、AWS AccessKey 等敏感凭据。所有配置必须通过 `app/core/config.py` 的 `pydantic-settings` 环境变量安全加载。 |
| **8** | **严禁裸报 500 异常与数据库内部堆栈给客户端** | 业务异常必须继承自 `AppException`；未捕获系统异常必须统一被全局中间件/Exception Handler 拦截，包装为标准错误信封（`code`, `message`, `data`, `traceId`, `isRetryable`），严禁泄漏数据库表名与底层错误栈。 |
| **9** | **严禁使用浮点数 `float` 计算或存储货币金额** | 涉及交易、支付、退款及账户余额等金额计算与模型字段，严禁使用 `float`（防止二进制浮点精度丢失）。必须统一使用最小货币单位整数（如“分”，`int`）或标准高精度 `Decimal`。 |
| **10** | **严禁交付未通过本地质量门禁的代码** | 代码编写完毕后，必须运行格式化与静态检查（`ruff check .`、`ruff format --check .`、`mypy .`），并确保全量异步单元测试（`pytest`）0 报错通过。 |

---

## 4. 自主权与权限边界

- **AI 可以自主执行的操作**：
  - 在 `app/domains/<domain>/` 下创建或修改 Models、Schemas、Repositories、Services、Routers 及单元测试。
  - 在 `app/shared/` 与 `app/core/` 下添加无业务属性的基础工具类、异常定义与依赖注入 Provider。
  - 运行 `alembic revision --autogenerate -m "..."` 生成迁移脚本并完善检查逻辑。
  - 运行本地质量门禁命令（`ruff`、`mypy`、`pytest`）。
- **AI 必须征得开发者明确同意后才可执行的操作**：
  - 在 `pyproject.toml` 中新增或删除第三方依赖包。
  - 修改 `app/core/config.py`、`alembic/env.py` 等底层核心配置。
  - 编写或执行删除数据库表、清空表数据或破坏性非兼容字段迁移的 DDL。
  - 使用 `# type: ignore` 或 `# noqa` 暴力屏蔽代码规范警告。
