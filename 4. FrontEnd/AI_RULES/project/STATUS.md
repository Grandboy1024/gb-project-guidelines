# 项目当前进展与状态 (STATUS)

规范编号：PROJECT-003  
版本：1.0.0  
适用范围：强制执行  

---

## 1. 当前阶段：架构基线与 AI 规范体系建设完毕

React Web 运营管理后台工程已完成架构选型对齐、基础规约制定，并已按照 `9. AI/AI 开发协作规范.md` 与 `4. FrontEnd/React 项目规范.md` 完整确立了 AI 协作规则体系。

---

## 2. 核心架构与基础设施就绪矩阵

| 架构分层 / 模块 | 建设状态 | 实施详情与核心标准 |
|---|---|---|
| **架构总设计** | 已就绪 (Completed) | 领域就近聚合 (Colocation)、TanStack Query 服务端缓存与 Zustand 客户端分离已固化 |
| **AI 协作规范体系** | 已就绪 (Completed) | `RULES.md` 总清单、`core/`、`development/`、`project/`、平台适配入口全部就绪 |
| **服务端状态管理** | 已就绪 (Completed) | TanStack Query v5 规范确立，QueryKey 结构化数组管理与主动失效机制已建立 |
| **客户端轻量状态** | 已就绪 (Completed) | Zustand 规范确立，严格限制仅管理侧边栏、主题模式与当前登录用户信息 |
| **数据契约与校验** | 已就绪 (Completed) | Zod Schema 运行时校验与 TypeScript 自动类型推导机制已确立 |
| **UI 与交互规范** | 已就绪 (Completed) | Ant Design 5.x 主题 Tokens、四态闭环渲染 (Loading/Empty/Error/Success) 与长文本截断标准已制定 |
| **网络层标准** | 已就绪 (Completed) | Axios 统一响应解包、Bearer Token 自动挂载、401 拦截重定向与取消令牌支持已定义 |
| **工程化工具基石** | 已就绪 (Completed) | Vite + TypeScript 严格模式 + ESLint + Prettier + Vitest 测试套件已就绪 |

---

## 3. 后续业务迭代路线图 (Roadmap)

1. **业务功能实施阶段**：
   - 落地 `domains/auth`（运营登录、权限校验、Token 自动续期与退出状态机）。
   - 落地 `domains/devices`（智能灯设备虚拟化大盘、属性配置抽屉、实时日志控制台）。
   - 落地 `domains/ota`（固件文件拖拽验签上传、升级任务批次发布看板与进度统计）。
   - 落地 `domains/products`（智能灯具型号录入、物模型配置矩阵）。
2. **自动化流水线完善**：
   - 建立本地 Git Hook 与 GitHub Actions CI 流水线校验（`pnpm lint`、`tsc --noEmit`、`pnpm test`）。
   - 补充 Playwright 端到端关键链路冒烟测试。
