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
   - 凡是在回复中提及的文件，必须使用标准 `file://` 格式的 Markdown 链接，如 `[文件名.swift](file:///absolute/path/to/file.swift#L10-L25)`。
   - 链接文本统一使用文件基础名称（Basename），保持排版易读。
3. **严格并发与构建状态透明化**：
   - 简要汇报 Swift 严格并发检查结果与编译通过情况。
4. **决策与权衡重点说明**：
   - 涉及重要技术选型或架构取舍（如为何选用 `ObservableObject` + `@StateObject` 兼容 iOS 16、为何在特定场景采用结构化 `Task` 取消），简要说明设计理由与防御考量。

---

## 2. 任务交付标准汇报模版

当完成功能开发、重构或 Bug 修复任务后，按照以下格式向开发者交付：

```markdown
### 1. 任务完成概括
- 简要总结完成了哪些 iOS 模块、SwiftUI 页面或功能的开发与重构。

### 2. 变更文件清单
- [XxxRequest.swift](file:///path/to/XxxRequest.swift) - 新增 API 请求对象，声明响应泛型与请求体。
- [XxxRepository.swift](file:///path/to/XxxRepository.swift) - 业务仓储协议与网络调度实现。
- [XxxViewModel.swift](file:///path/to/XxxViewModel.swift) - 基于 ObservableObject 与 @MainActor 的页面控制器。
- [XxxView.swift](file:///path/to/XxxView.swift) - SwiftUI 视图组件，完整处理四态闭环与 44pt 触控防御。
- [AppRoute.swift](file:///path/to/AppRoute.swift) - 注册强类型 AppRoute 路由枚举。
- [Xxx.xcstrings](file:///path/to/Xxx.xcstrings) - 本模块中英文国际化配置。

### 3. 工具链与质量门禁验证
- `SwiftLint`: 静态检查通过，0 警告 / 0 错误。
- `Strict Concurrency`: 通过，无数据竞态与线程安全警告。
- `Swift Testing`: 通过，X 个单元测试全绿通过。

### 4. 关键防御性实现说明
- 简要列出针对该页面实施的防御性设计（如 44pt 触控热区、长文本行数截断、加载态按钮禁用、防键盘遮挡、Task 退出取消机制等）。
```
