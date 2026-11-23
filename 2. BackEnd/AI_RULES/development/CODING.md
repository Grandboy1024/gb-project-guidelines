# Python & FastAPI 编码与防御规范 (CODING)

规范编号：DEV-001  
版本：1.0.0  
适用范围：强制执行  

---

## 1. Modern Python 3.12+ 强类型规范

1. **类型注解与静态安全**：
   - 所有的函数入参、返回值以及复杂数据结构必须显式声明类型注解。
   - 采用 Python 3.10+ 原生语法：使用 `A | B` 替代 `Union[A, B]`，使用 `list[str]`、`dict[str, Any]` 替代 `typing.List`、`typing.Dict`。
   - 泛型仓储统一采用 PEP 695 语法或 `TypeVar` 声明：
     ```python
     ModelType = TypeVar("ModelType", bound=DeclarativeBase)
     ```
2. **严格禁止裸写 `Any`**：
   - 除极为特殊的序列化边界外，严禁通篇使用 `Any` 逃避静态检查。

---

## 2. Pydantic V2 契约与序列化

1. **统一继承与配置**：
   - 所有 Schema 继承自自定义 `BaseSchema`（统一继承 `pydantic.BaseModel`）。
   - 涉及从 SQLAlchemy ORM 模型转换的 Response Schema，必须显式配置：
     ```python
     class UserResponse(BaseSchema):
         id: str
         email: str
         nickname: str
         is_active: bool
         created_at: datetime

         model_config = ConfigDict(from_attributes=True, populate_by_name=True)
     ```
2. **字段命名与序列化风格 (camelCase 契约)**：
   - Python 代码内部属性遵循 `snake_case`。
   - 对外 HTTP API 统一使用 `camelCase` 驼峰风格。在 `BaseSchema` 中配置 `alias_generator = to_camel`，自动完成双向转换。
3. **输入参数的边界校验**：
   - 字符串必须指定长度限制（`Field(min_length=1, max_length=128)`）。
   - 分页入参必须限制上界（`pageSize: int = Field(default=20, ge=1, le=100)`）。
   - 禁止在参数校验阶段放过任何非法数据类型。

---

## 3. 统一响应契约与异常处理

### 3.1 成功响应统一信封

所有标准 RESTful 接口正常返回结构约定为：

```json
{
  "code": 0,
  "message": "success",
  "data": { ... },
  "traceId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "timestamp": "2026-11-23T08:00:00.123Z"
}
```

### 3.2 异常体系与统一错误信封

1. **领域异常密封类 (`AppException`)**：
   - 业务逻辑中的所有错误必须继承自统一基类：
     ```python
     class AppException(Exception):
         def __init__(
             self,
             code: int,
             message: str,
             status_code: int = 400,
             data: Any = None,
             is_retryable: bool = False
         ):
             self.code = code
             self.message = message
             self.status_code = status_code
             self.data = data
             self.is_retryable = is_retryable
     ```
2. **全局异常拦截器 (Exception Handlers)**：
   - 在 `app/main.py` 中注册 `AppException`、`RequestValidationError` 与 `Exception` 统一捕获器。
   - 严禁将数据库原始报错（如 `UniqueViolation`、`ForeignKeyViolation`）直接抛给前端，必须统一转换为 `409 Conflict` 或 `400 Bad Request` 业务错误。

---

## 4. 并发安全与高防御性编程

1. **热点业务原子扣减与行级锁**：
   - 在商品库存扣减、优惠券核销、设备绑定抢占等场景，必须使用数据库行级排他锁：
     ```python
     async def get_for_update(self, session: AsyncSession, entity_id: str) -> Entity | None:
         stmt = select(Entity).where(Entity.id == entity_id).with_for_update()
         result = await session.execute(stmt)
         return result.scalar_one_or_none()
     ```
2. **货币与金融高精度计算**：
   - 严禁使用 `float` 进行金额计算。
   - 数据库存储使用整型最小单位（`integer`，存储分）或高精度 `Numeric(18, 4)`；Python 中使用 `int` 或 `Decimal`。
3. **SQLAlchemy 异步事务最佳实践**：
   - 复合业务操作必须显式包裹在事务中：
     ```python
     async with session.begin():
         await order_repo.create(session, order)
         await inventory_repo.deduct(session, product_id, count)
     ```
4. **日志与链路追踪 (Structlog)**：
   - 严禁裸写 `print()`。
   - 统一使用 `structlog`，通过 ASGI 中间件从 HTTP Header 提取或生成 `X-Trace-Id` 并绑定到 `contextvars`：
     ```python
     logger = structlog.get_logger()
     logger.info("order_created", order_id=order.id, user_id=user.id, amount=order.amount)
     ```
   - 密码、Token、手机号等关键个人隐私字段必须做脱敏保护。
