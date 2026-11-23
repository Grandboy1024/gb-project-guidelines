# 架构决策记录 (DECISIONS)

规范编号：PROJECT-002  
版本：1.0.0  
适用范围：强制执行  

---

## ADR-001: 采用 Vite + React 18/19 + TypeScript 作为核心构建中枢

- **决策状态**：已确认 (Confirmed)
- **决策背景**：传统的 Webpack 构建速度慢、热更新延迟大；JavaScript 缺乏类型约束导致运行时错误高发。
- **决策结论**：全工程统一采用 **Vite** 作为构建工具，搭配 **React 18/19** 与 **TypeScript 5.x**。
- **强制规则**：
  - 开启 `strict: true`，严禁在生产代码中使用 `any`。

---

## ADR-002: 采用 TanStack Query v5 作为唯一服务端异步状态管理方案

- **决策状态**：已确认 (Confirmed)
- **决策背景**：使用 Redux 或 Zustand 存储服务端接口数据需要手写大量的 loading/error 样板代码，且难以处理缓存失效、垃圾回收与窗口聚焦重新请求。
- **决策结论**：全项目统一采用 **TanStack Query (React Query) v5** 托管全部由后端 API 返回的异步状态。
- **强制规则**：
  - 严禁手动 `set` 服务端数据到全局 Store。
  - 数据变更必须通过 `useMutation` 搭配 `invalidateQueries` 实现缓存自动刷新。

---

## ADR-003: 采用 Zustand 管理轻量客户端会话与 UI 状态

- **决策状态**：已确认 (Confirmed)
- **决策背景**：Redux 架构繁琐臃肿；React Context 在高频更新下易引发全树不必要重绘。
- **决策结论**：全项目统一采用 **Zustand** 管理纯客户端轻量状态。
- **强制规则**：
  - 仅用于管理主题模式、侧边栏展开折叠、弹窗显隐与当前登录用户信息。

---

## ADR-004: 采用 Ant Design 5.x 与 Design Tokens 规范

- **决策状态**：已确认 (Confirmed)
- **决策背景**：自研完整 B 端 UI 组件库成本高昂，且难以保障无障碍与交互一致性。
- **决策结论**：统一采用 **Ant Design 5.x** 搭配 Design Tokens，辅以 **Tailwind CSS** 处理微观布局。
- **强制规则**：
  - 色彩与间距一律读取主题 Tokens，严禁手写奇数魔法像素与硬编码十六进制色值。

---

## ADR-005: 采用 Zod 驱动契约校验与类型推导

- **决策状态**：已确认 (Confirmed)
- **决策背景**：后端接口变更容易导致前端出现 `undefined is not an object` 等致命白屏，仅依靠 TypeScript 编译期检查无法拦截运行期数据异常。
- **决策结论**：统一采用 **Zod** 定义运行时 Schema 并自动推导静态类型。
- **强制规则**：
  - 复杂表单与核心 API 响应统一通过 Zod Schema 进行校验。

---

## ADR-006: 采用领域就近聚合 (Colocation) 目录组织

- **决策状态**：已确认 (Confirmed)
- **决策背景**：传统的按文件角色分类（所有 pages 放在一起、所有 api 放在一起）导致修改一个功能需要在深层目录树中频繁跳跃。
- **决策结论**：采用 **按业务领域组织 (Domain Colocation)**，每个 Domain 包含自己的 `api`、`components`、`hooks`、`pages`、`schemas`、`types`。
- **强制规则**：
  - 领域内部细节私有，仅通过 `index.ts` 对外暴露公开页面与组件。

---

## ADR-007: 采用集中化路由常量与 AuthGuard 路由守卫

- **决策状态**：已确认 (Confirmed)
- **决策背景**：裸写魔术字符串路径会导致重构困难；未受保护的路由易引发越权与闪烁。
- **决策结论**：路由常量集中在 `src/app/router/paths.ts` 中管理，统一使用 `AuthGuard` 拦截未登录访问。
- **强制规则**：
  - 严禁硬编码路径字符串跳转；未登录统一携带 `redirect` 参数跳转登录页。

---

## ADR-008: 采用统一响应外壳解包与错误拦截机制

- **决策状态**：已确认 (Confirmed)
- **决策背景**：后端统一返回 `{ code: 0, data: ..., message: ... }` 格式，如果在每个组件内重复解包和弹窗提示，会产生大量冗余代码。
- **决策结论**：网络层 `client.ts` 统一拦截并解包 `data`，非 0 业务异常统一抛出 `ApiClientError` 并轻量提示。
- **强制规则**：
  - 业务代码直接接收纯净 `data` 数据，401 自动触发登出重定向。
