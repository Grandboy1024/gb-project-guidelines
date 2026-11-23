# AI 沟通与交互规范 (COMMUNICATION)

规范编号：CORE-003  
版本：1.0.0  
适用范围：强制执行  

---

## 1. 核心沟通原则

1. **精准精炼、结果导向**：
   - 杜绝多余的寒暄与废话，以条理清晰的技术语言直接回应。
   - 采用标准 GitHub Flavored Markdown 组织内容。
2. **所有文件必须提供可点击的 Markdown 链接**：
   - 凡是在回复中提及的文件，必须使用标准 `file://` 格式的 Markdown 链接，如 `[文件名.py](file:///absolute/path/to/file.py#L10-L25)`。
   - 链接文本统一使用文件基础名称（Basename），保持排版易读。
3. **命令执行透明化**：
   - 执行 CLI 命令（如 `alembic revision`、`ruff check`、`pytest`）后，简要汇报命令结果及生成产物状态。
4. **决策与权衡重点说明**：
   - 涉及重要技术选型或架构取舍（如为何选择 `selectinload` 而非 `joinedload`、为何选用行级悲观锁而不是乐观版本号），简要说明设计理由与防御考量。

---

## 2. 任务交付标准汇报模版

当完成功能开发、重构或 Bug 修复任务后，按照以下格式向开发者交付：

```markdown
### 1. 任务完成概括
- 简要总结完成了哪些模块、接口或业务链路的开发与优化。

### 2. 变更文件清单
- [models.py](file:///path/to/models.py) - 新增 Order 与 OrderItem ORM 模型。
- [xxxx_add_orders.py](file:///path/to/alembic/versions/xxxx.py) - Alembic 数据库版本迁移脚本。
- [repository.py](file:///path/to/repository.py) - OrderRepository 实现，支持带行锁的原子库存核销。
- [service.py](file:///path/to/service.py) - 订单核心编排服务与事务边界控制。
- [schemas.py](file:///path/to/schemas.py) - Pydantic V2 输入输出契约定义。
- [router.py](file:///path/to/router.py) - /api/v1/orders 接口路由挂载。
- [test_orders.py](file:///path/to/test_orders.py) - 单元测试与并发安全性用例。

### 3. 工具链与质量门禁验证
- `alembic upgrade head`: 成功迁移至最新版本。
- `ruff check .`: 通过，0 错误 / 0 警告。
- `mypy .`: 通过，0 类型错误。
- `pytest`: 通过，X 个测试用例全绿通过。

### 4. 关键防御性实现说明
- 简要列出针对该任务实施的防御性设计（如行级排他锁防超卖、SQLAlchemy 预加载防 N+1、统一错误信封封装与敏感字段脱敏等）。
```

---

## 3. 歧义与冲突处理机制

1. **架构冲突纠正**：
   - 若开发者的指令与项目核心架构规范相违背（例如要求“直接在 Router 里写 SQL 查询数据库”或“在端点里用 requests 调外部服务”），**绝对不能盲从**。
   - 必须以礼貌专业的态度指出该做法违背了规范 (`CORE-001`, `DEV-002`)，并主动给出符合分层架构和异步非阻塞的标准实现方案供选择。
2. **协议与数据结构未定义**：
   - 若遇到未明确的字段定义、外键级联策略或错误码，不得凭空编写猜测性的表字段，应主动列出所需确认项并与开发者对齐。
3. **高危与破坏性操作**：
   - 在删除数据表、执行不可逆字段变更、清空 Redis 缓存或修改全局环境配置前，必须向开发者明确说明潜在影响并获得确认。
