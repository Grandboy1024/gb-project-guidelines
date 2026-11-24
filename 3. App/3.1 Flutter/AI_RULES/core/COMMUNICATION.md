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
   - 凡是在回复中提及的文件，必须使用标准 `file://` 格式的 Markdown 链接，如 `[文件名.dart](file:///absolute/path/to/file.dart#L10-L25)`。
   - 链接文本统一使用文件基础名称（Basename），保持排版易读。
3. **命令执行透明化**：
   - 执行 CLI 命令（如 `build_runner`、`gen_l10n`、`flutter analyze`）后，简要汇报命令结果及生成产物状态。
4. **决策与权衡重点说明**：
   - 涉及重要技术选型或架构取舍（如为何选择 `AsyncNotifier<void>` 而非复杂的 State 类），简要说明设计理由与防御考量。

---

## 2. 任务交付标准汇报模版

当完成功能开发、重构或 Bug 修复任务后，按照以下格式向开发者交付：

```markdown
### 1. 任务完成概括
- 简要总结完成了哪些模块、页面或功能的开发与重构。

### 2. 变更文件清单
- [xxx_request.dart](file:///path/to/xxx_request.dart) - 新增 API 请求对象，支持 CancelToken。
- [xxx_repository.dart](file:///path/to/xxx_repository.dart) - 业务仓储与生成的 Provider。
- [xxx_controller.dart](file:///path/to/xxx_controller.dart) - 基于 AsyncNotifier 的页面控制器。
- [xxx_page.dart](file:///path/to/xxx_page.dart) - 视图层组件，完整处理四态与输入框交互。
- [app_routes.dart](file:///path/to/app_routes.dart) - 注册强类型 GoRoute 路由。
- [xxx_en.arb](file:///path/to/xxx_en.arb) & [xxx_zh.arb](file:///path/to/xxx_zh.arb) - 本模块中英文国际化配置。

### 3. 工具链与质量门禁验证
- `dart run build_runner build -d`: 成功，生成 X 个关联文件。
- `dart run tool/gen_l10n.dart`: 成功，生成全局及当前 feature 的本地化代理代码。
- `flutter analyze`: 通过，0 警告 / 0 错误。
- `flutter test`: 通过，X 个测试用例全绿通过。

### 4. 关键防御性实现说明
- 简要列出针对该页面实施的防御性设计（如长文本截断、加载中按钮禁用、软键盘点击收起、AlwaysScrollable 下拉刷新等）。
```

---

## 3. 歧义与冲突处理机制

1. **架构冲突纠正**：
   - 若开发者的指令与项目核心架构规范相违背（例如要求“在这个页面使用 GetX 弹窗”或“把 BuildContext 传给 Controller”），**绝对不能盲从**。
   - 必须以礼貌专业的态度指出该做法违背了规范 (`CORE-001`, `DEV-002`)，并主动给出符合 Riverpod + GoRouter 的标准实现方案供选择。
2. **接口协议未定义**：
   - 若遇到未明确的后端接口字段或格式，不得凭空编写猜测性的数据类，应主动列出所需字段清单并与开发者确认。
3. **高危与破坏性操作**：
   - 在删除文件、清空缓存、修改底层共享模型或变更全局主题 Token 前，必须向开发者明确说明影响范围并获得确认。
