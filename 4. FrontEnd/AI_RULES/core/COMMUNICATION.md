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
   - 凡是在回复中提及的文件，必须使用标准 `file://` 格式的 Markdown 链接，如 `[文件名.tsx](file:///absolute/path/to/file.tsx#L10-L25)`。
   - 链接文本统一使用文件基础名称（Basename），保持排版易读。
3. **命令执行透明化**：
   - 执行 CLI 命令（如 `pnpm lint`、`tsc`、`pnpm test`）后，简要汇报命令结果及生成产物状态。
4. **决策与权衡重点说明**：
   - 涉及重要技术选型或架构取舍（如为何将过滤参数放入 URL Query 而非 Zustand、为何使用 `useMutation` 配合 `invalidateQueries`），简要说明设计理由与防御考量。

---

## 2. 任务交付标准汇报模版

当完成功能开发、重构或 Bug 修复任务后，按照以下格式向开发者交付：

```markdown
### 1. 任务完成概括
- 简要总结完成了哪些模块、页面或 UI 组件的开发与优化。

### 2. 变更文件清单
- [device.schema.ts](file:///path/to/device.schema.ts) - Zod 契约定义与 TypeScript 导出类型。
- [device.api.ts](file:///path/to/device.api.ts) - 设备领域 API 请求函数。
- [useDevices.ts](file:///path/to/useDevices.ts) - TanStack Query 封装，支持自动缓存失效与乐观更新。
- [DeviceListPage.tsx](file:///path/to/DeviceListPage.tsx) - 列表页面组件，完整实现四态闭环与表格分页。
- [paths.ts](file:///path/to/paths.ts) - 注册类型安全的路由常量。
- [DeviceListPage.test.tsx](file:///path/to/DeviceListPage.test.tsx) - 单元测试用例。

### 3. 工具链与质量门禁验证
- `pnpm exec tsc --noEmit`: 通过，0 类型错误。
- `pnpm lint`: 通过，0 警告 / 0 错误。
- `pnpm test`: 通过，X 个测试用例全绿通过。

### 4. 关键防御性实现说明
- 简要列出针对该页面实施的防御性设计（如表格长文本 Tooltip 省略、搜索输入 300ms 防抖、表单提交中按钮 loading 禁用、动态列表使用实体 id 作为 key 等）。
```

---

## 3. 歧义与冲突处理机制

1. **架构冲突纠正**：
   - 若开发者的指令与项目核心架构规范相违背（例如要求“把从 API 拉下来的用户列表直接存到全局 Redux/Zustand 里”或“在循环列表里用 index 作 key”），**绝对不能盲从**。
   - 必须以礼貌专业的态度指出该做法违背了规范 (`CORE-001`, `DEV-002`)，并主动给出符合 TanStack Query 缓存管理的标准实现方案供选择。
2. **接口协议未定义**：
   - 若遇到未明确的后端接口字段或数据类型，不得凭空编写猜测性的数据契约，应主动列出所需字段清单并与开发者确认。
3. **高危与破坏性操作**：
   - 在升级第三方包大版本、重构跨 Domain 共享组件或修改全局主题 Token 前，必须向开发者明确说明影响范围并获得确认。
