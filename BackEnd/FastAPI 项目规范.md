```jsx
Project/
├── .github/
│   ├── workflows/
│   │   └── ci.yml                     # 自动化检查 (Ruff, Mypy, Pytest)
│   └── pull_request_template.md
├── alembic/                           # 数据库版本迁移
│   ├── versions/                      # 具体迁移脚本
│   ├── env.py                         # 注册各 Domain Model 的 Metadata
│   └── script.py.mako
├── app/
│   ├── __init__.py
│   ├── main.py                        # 应用入口：生命周期 (Lifespan)、全局中间件、路由挂载
│   ├── core/                          # [无业务属性] 系统底层运行支撑
│   │   ├── __init__.py
│   │   ├── config.py                  # pydantic-settings 环境配置
│   │   ├── security.py                # 纯底层密码哈希算法、JWT encode/decode
│   │   ├── logging.py                 # 结构化日志 (Structlog/JSON) 与 TraceId 拦截
│   │   └── exceptions.py              # 核心基类异常与全局异常拦截器
│   │
│   ├── shared/                        # [跨域通用] 框架级公共基类与协议
│   │   ├── __init__.py
│   │   ├── db/
│   │   │   ├── __init__.py
│   │   │   ├── base.py                # DeclarativeBase, 通用 AuditMixin (ID, timestamps)
│   │   │   └── session.py             # AsyncEngine, sessionmaker 工厂
│   │   ├── base_repository.py         # 泛型通用 CRUD 基类
│   │   ├── base_schema.py             # 通用分页入参、统一 API 响应包装
│   │   └── deps.py                    # 纯技术依赖 (如 get_db, get_redis, 分页参数解析)
│   │
│   └── domains/                       # 【核心】业务特性/领域边界 (也可叫 modules/features)
│       ├── __init__.py
│       │
│       ├── auth/                      # 认证与授权上下文 (Identity & Access)
│       │   ├── __init__.py
│       │   ├── router.py              # /api/v1/auth/login, /refresh
│       │   ├── schemas.py             # LoginRequest, TokenResponse
│       │   ├── service.py             # 密码比对、Token 签发、黑名单校验
│       │   └── deps.py                # 跨域被广泛引用的业务依赖: get_current_user, require_roles
│       │
│       ├── users/                     # 用户领域 (Profile, Status)
│       │   ├── __init__.py
│       │   ├── models.py              # User ORM 模型
│       │   ├── schemas.py             # UserCreate, UserUpdate, UserProfileResponse
│       │   ├── repository.py          # UserRepository: 针对 users 表的特定持久化查询
│       │   ├── service.py             # 用户注册、禁用、修改资料业务规则
│       │   ├── router.py              # /api/v1/users/* 路由端点
│       │   └── exceptions.py          # UserNotFoundError, UserAlreadyExistsError
│       │
│       ├── products/                  # 商品领域
│       │   ├── __init__.py
│       │   ├── models.py              # Product, Category ORM
│       │   ├── schemas.py             # ProductCreate, ProductDetailResponse
│       │   ├── repository.py          # ProductRepository
│       │   ├── service.py             # 商品上架、库存变动核算
│       │   ├── router.py              # /api/v1/products/*
│       │   └── exceptions.py          # ProductOutOfStockError
│       │
│       └── orders/                    # 订单聚合域 (跨域业务协同典型)
│           ├── __init__.py
│           ├── models.py              # Order, OrderItem ORM
│           ├── schemas.py             # OrderCreateRequest, OrderDetailResponse
│           ├── repository.py          # OrderRepository
│           ├── service.py             # 核心编排：校验商品库存 -> 扣减 -> 落单 -> 触发支付
│           ├── router.py              # /api/v1/orders/*
│           ├── deps.py                # 订单专属依赖 (如 valid_order_id_or_404)
│           └── exceptions.py          # OrderStatusInvalidError
│
├── tests/                             # 测试目录（镜像 domains 结构）
│   ├── conftest.py                    # 全局 TestClient, AsyncSession, 内存 DB Fixture
│   ├── shared/                        # 测试公共库/工具
│   ├── domains/
│   │   ├── auth/
│   │   │   ├── test_service.py
│   │   │   └── test_api.py
│   │   ├── users/
│   │   │   └── test_repository.py
│   │   └── orders/
│   │       ├── test_flow.py           # 跨域链路集成测试
│   │       └── test_api.py
│   └── e2e/                           # 真正的黑盒端到端验收用例
│
├── .editorconfig
├── .env.example
├── .gitignore
├── .pre-commit-config.yaml
├── .python-version                    # Python 运行时声明 (如 3.12)
├── alembic.ini                        # Alembic 入口配置
├── docker-compose.yml                 # 本地基础设施编排 (Postgres, Redis)
├── Dockerfile                         # 多阶段容器构建
├── Makefile                           # 统一开发 CLI (make run, make test, make lint)
├── pyproject.toml                     # 统一管理项目元数据、依赖、ruff/pytest/mypy 规则
├── uv.lock                            # 确定性依赖锁定文件
└── README.md

```