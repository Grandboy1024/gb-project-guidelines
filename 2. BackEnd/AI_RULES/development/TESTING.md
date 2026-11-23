# 质量保证与测试规范 (TESTING)

规范编号：DEV-003  
版本：1.0.0  
适用范围：强制执行  

---

## 1. 测试体系与分层模型

本项目以后端高可用、业务逻辑零缺陷为核心目标，建立全面的自动化测试体系：

```
        / \
       /   \      E2E 端到端冒烟与跨服务测试
      /-----\
     /       \    API 集成测试 (httpx.AsyncClient + 测试 DB)
    /---------\
   /           \  Unit 单元测试 (Service 业务规则 & Repository 查询)
  /-------------\
```

- **仓储层测试 (Repository Tests)**：针对真实的异步测试数据库（PostgreSQL/Docker），验证 ORM 映射、自定义 SQL 条件、事务与预加载。
- **服务层测试 (Service Tests)**：Mock 数据仓储与外部接口，验证纯业务规则编排、计算逻辑与异常熔断分支。
- **接口集成测试 (API Tests)**：使用 `httpx.AsyncClient`，验证路由端点、入参验证、权限拦截、状态码与统一错误信封。
- **数据库迁移测试 (Migration Tests)**：验证 Alembic 迁移脚本在空库上的双向可逆性（`upgrade head` -> `downgrade base`）。

---

## 2. 源码与测试目录镜像规范

测试文件必须严格镜像 `app/` 目录结构放置在 `tests/` 下：

```
app/domains/orders/service.py
└── tests/domains/orders/test_service.py

app/domains/orders/router.py
└── tests/domains/orders/test_api.py
```

---

## 3. 异步测试标准范式 (`pytest-asyncio` & `httpx`)

### 3.1 全局 Fixtures 配置 (`tests/conftest.py`)

```python
import pytest
import pytest_asyncio
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from app.main import app
from app.shared.deps import get_db
from app.shared.db.base import Base

TEST_DATABASE_URL = "postgresql+asyncpg://postgres:postgres@localhost:5432/test_db"

@pytest_asyncio.fixture(scope="session")
async def test_engine():
    engine = create_async_engine(TEST_DATABASE_URL, echo=False)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()

@pytest_asyncio.fixture
async def db_session(test_engine) -> AsyncSession:
    session_factory = async_sessionmaker(test_engine, expire_on_commit=False)
    async with session_factory() as session:
        yield session
        await session.rollback()

@pytest_asyncio.fixture
async def client(db_session: AsyncSession) -> AsyncClient:
    async def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac
    app.dependency_overrides.clear()
```

### 3.2 API 集成测试用例标准范式

```python
# tests/domains/auth/test_api.py
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_login_success(client: AsyncClient, db_session):
    # 准备测试数据...
    response = await client.post(
        "/api/v1/auth/login",
        json={"email": "test@example.com", "password": "SecurePassword123!"}
    )
    assert response.status_code == 200
    data = response.json()
    assert data["code"] == 0
    assert "accessToken" in data["data"]
    assert "traceId" in data

@pytest.mark.asyncio
async def test_login_invalid_password(client: AsyncClient):
    response = await client.post(
        "/api/v1/auth/login",
        json={"email": "test@example.com", "password": "WrongPassword"}
    )
    assert response.status_code == 400
    data = response.json()
    assert data["code"] != 0
    assert data["isRetryable"] is False
```

---

## 4. 本地提交与 CI 质量门禁套件

任何代码提交或任务交付前，必须在工程根目录下执行并通过以下三道质量门禁：

```bash
# 1. 代码质量与格式化检查
ruff check . && ruff format --check .

# 2. 严格静态类型检查
mypy .

# 3. 运行全量异步自动化测试套件
pytest -v
```
