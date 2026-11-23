# 业务背景与工程全景 (CONTEXT)

规范编号：PROJECT-001  
版本：1.0.0  
适用范围：强制执行  

---

## 1. 产品概述与设计目标

本项目是基于 **Python 3.12+** 与 **FastAPI** 构建的高性能异步云端业务服务中枢与运营管理后台 API。

系统的核心定位是面向智能灯具移动客户端 (Flutter App)、运营管理后台 (React Web) 以及物联网消息中枢 (MQTT/EMQX) 的核心业务大脑，核心设计目标包括：
1. **高性能与全异步非阻塞**：基于 ASGI 异步驱动架构，以低资源开销支撑数万并发连接与高频设备状态同步。
2. **领域边界清晰的模块化单体**：初期坚决避免过早微服务化带来的分布式事务与运维复杂度，采用领域模块化单体（Modular Monolith）实现高内聚低耦合。
3. **企业级高鲁棒性与金融级防御**：对关键库存、订单并发进行悲观锁与幂等保护，杜绝超卖与数据竞争；全链路注入分布式追踪标识 `traceId`。

---

## 2. 核心业务功能模块全景

所有业务领域代码统一组织在 `app/domains/` 目录下，各模块完全自闭环：

```
app/domains/
├── auth/           # 认证与授权上下文 (JWT 签发、刷新、黑名单、角色权限 RBAC)
├── users/          # 用户中心 (用户信息管理、状态封禁、个人偏好)
├── products/       # 产品与型号管理 (智能灯硬件规格、灯具品类、功能定义)
├── orders/         # 交易与订单聚合域 (下单编排、库存原子校验、支付流转)
├── devices/        # 智能灯设备核心域 (设备激活、绑定/解绑、MQTT 状态同步与控制)
└── ota/            # 固件升级与设备管理 (固件上传、升级任务派发、进度追踪与差分校验)
```

---

## 3. 基础设施与跨模块共享服务

```
app/
├── core/                  # 与具体业务无关的纯系统底层运行支撑
│   ├── config.py          # pydantic-settings 集中配置（默认 prod，支持安全隔离）
│   ├── security.py        # 密码哈希算法 (Argon2/Bcrypt)、JWT encode/decode
│   ├── logging.py         # Structlog JSON 结构化日志与 TraceId 拦截配置
│   └── exceptions.py      # AppException 基础密封异常体系与全局拦截处理器
├── shared/                # 跨领域通用的框架级公共基类与协议
│   ├── db/
│   │   ├── base.py        # DeclarativeBase 与通用 AuditMixin (ID, timestamps, is_deleted)
│   │   └── session.py     # AsyncEngine 与 async_sessionmaker 工厂
│   ├── base_repository.py # 泛型通用 CRUD 仓储基类 (BaseRepository[Model])
│   ├── base_schema.py     # 通用分页入参、统一 API 响应包装契约
│   └── deps.py            # 纯技术依赖注入 (get_db, get_redis, 分页解析器)
└── main.py                # 应用入口：lifespan 生命周期、全局中间件、路由汇总挂载
```

---

## 4. 关键工程质量优先级

1. **强分层与依赖单向流动**：严禁在 Router 中直接执行 SQL 或调用数据库 Session；Router 必须通过 Service 调度。
2. **完全异步并发安全**：严禁在异步端点中使用阻塞性第三方库；热点扣减与设备抢占必须使用行级排他锁或分布式锁。
3. **数据库变更版本可控**：严禁手动修改数据库表结构，所有变更必须由 Alembic 迁移脚本承载并支持回滚。
