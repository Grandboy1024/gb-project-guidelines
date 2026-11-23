# AI 统一规范清单 (Rules Manifest)

版本：1.0.0  
适用项目：React Web 运营管理后台 (Web Frontend)  
最后更新：2026-11-23  

---

## 1. 规范定位与目的

本文档是本项目所有 AI 工具、智能助手（如 Cursor、Claude Code、GitHub Copilot、Gemini CLI、Antigravity、OpenDevin 等）的**唯一权威主入口**与规范索引清单。

所有 AI 在执行任何需求开发、缺陷修复、架构重构或测试编写任务前，**必须首先读取并严格遵守本清单及关联规范**。

本清单明确了各项规范的权威层级、任务适用矩阵以及冲突仲裁优先级。

---

## 2. 强制性核心规范 (Mandatory Core Rules)

以下文档对所有 AI 工具、所有开发场景均具有最高强制执行力：

| 规范编号 | 文件路径 | 核心内容概述 |
|---|---|---|
| **CORE-001** | [core/MASTER.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/4.%20FrontEnd/AI_RULES/core/MASTER.md) | AI 角色定位、行为准则与 10 条绝对红线禁令 |
| **CORE-002** | [core/WORKFLOW.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/4.%20FrontEnd/AI_RULES/core/WORKFLOW.md) | 5 阶段标准化作业流程 (SOP) 与任务执行规范 |
| **CORE-003** | [core/COMMUNICATION.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/4.%20FrontEnd/AI_RULES/core/COMMUNICATION.md) | 沟通规范、代码变更呈现与决策对齐机制 |

---

## 3. 开发规范 (Development Rules)

在进行 React / TypeScript 代码编写、页面还原、状态管理或测试时，必须严格遵守以下规范：

| 规范编号 | 文件路径 | 核心内容概述 |
|---|---|---|
| **DEV-001** | [development/CODING.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/4.%20FrontEnd/AI_RULES/development/CODING.md) | TypeScript 严格模式、Zod 契约、UI 防御性设计、四态闭环与交互规范 |
| **DEV-002** | [development/ARCHITECTURE.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/4.%20FrontEnd/AI_RULES/development/ARCHITECTURE.md) | 领域就近聚合 (Colocation)、TanStack Query 服务端缓存与 Zustand 客户端分离、路由守卫 |
| **DEV-003** | [development/TESTING.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/4.%20FrontEnd/AI_RULES/development/TESTING.md) | Vitest + React Testing Library 单元/Hook 测试、MSW 接口 Mock 与质量门禁 |

---

## 4. 项目知识库 (Project Knowledge)

以下文档提供项目的业务全景、历史技术决策与当前进展：

| 规范编号 | 文件路径 | 核心内容概述 |
|---|---|---|
| **PROJECT-001** | [project/CONTEXT.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/4.%20FrontEnd/AI_RULES/project/CONTEXT.md) | 业务背景、产品定位与核心领域模块划分 |
| **PROJECT-002** | [project/DECISIONS.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/4.%20FrontEnd/AI_RULES/project/DECISIONS.md) | 已确认的技术架构决策 (ADR-001 ~ ADR-008) |
| **PROJECT-003** | [project/STATUS.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/4.%20FrontEnd/AI_RULES/project/STATUS.md) | 项目当前进展阶段、已完成模块与后续演进目标 |

---

## 5. 核心技术选型速查表

AI 生成的所有代码必须 100% 契合以下技术决策，严禁违背：

| 技术领域 | 选型方案 | 对应规范索引 |
|---|---|---|
| **构建工具 & 框架** | `Vite` + `React 18/19` + `TypeScript 5.x` | `DEV-001` / `React 项目规范.md` 第一节 |
| **路由导航** | `React Router v6+` (路由守卫 AuthGuard + 布局嵌套 MainLayout) | `DEV-002` / `React 项目规范.md` 第五节 |
| **服务端状态管理** | `TanStack Query v5` (统一托管异步数据缓存、重新拉取、分页) | `DEV-002` / `React 项目规范.md` 第四节 |
| **客户端全局状态** | `Zustand` (仅限轻量非服务端数据：主题、侧边栏折叠、当前用户信息) | `DEV-002` / `React 项目规范.md` 第四节 |
| **UI 与组件库** | `Ant Design 5.x` + `Tailwind CSS` (原子样式辅助) | `DEV-001` / `React 项目规范.md` 第七节 |
| **数据契约与校验** | `Zod` (用于 API 响应运行时校验与表单验证) | `DEV-001` / `React 项目规范.md` 第四节 |
| **网络请求库** | 封装后的 `Axios` / `client.ts` (统一错误解包、拦截与取消) | `DEV-002` / `React 项目规范.md` 第六节 |
| **环境配置** | `core/config/env.ts` (基于 Vite `import.meta.env` + Zod 校验) | `DEV-001` / `React 项目规范.md` 第八节 |
| **国际化方案** | `i18next` + `react-i18next` (按业务领域拆分 locale json) | `DEV-002` / `React 项目规范.md` 第九节 |
| **测试框架** | `Vitest` + `@testing-library/react` + `MSW` | `DEV-003` / `React 项目规范.md` 第十节 |
| **包管理与代码规范** | `pnpm` + `ESLint 9+` + `Prettier` (开启严格 TS 规则) | `DEV-003` / `React 项目规范.md` 第十节 |

---

## 6. 执行准则与约束

1. **先读后写**：在编写或修改任何代码前，必须先查阅本清单并定位具体规范。
2. **状态分离铁律**：严禁将服务端 API 数据手工同步存入 Zustand 或 Redux；必须统一由 TanStack Query 托管。
3. **禁止擅自引入替代库**：严禁在 `package.json` 中擅自引入 Redux、MobX、jQuery 等非项目既定方案。
4. **四态闭环防御**：所有异步列表与数据页面必须完整处理 Loading、Empty、Error（带重试按钮）与 Success 四态。
5. **及时报告缺失**：若发现引用的规范文件不存在，必须第一时间向开发者报告。

---

## 7. 文档冲突优先级

当不同指令或文档出现冲突时，严格按照以下优先级从高到低仲裁：

1. 系统与平台安全、运行环境与生产打包边界约束。
2. 开发者针对当前任务给出的显式指令。
3. 已确认的架构决策记录 (`project/DECISIONS.md`)。
4. 核心行为准则与红线 (`core/MASTER.md`, `core/WORKFLOW.md`)。
5. 任务相关的开发规范 (`development/CODING.md`, `ARCHITECTURE.md`, `TESTING.md`)。
6. 项目背景与状态记录 (`project/CONTEXT.md`, `STATUS.md`)。

---

## 8. 规范维护机制

本文档是项目规范的最高登记处。
当新增、移除或调整任何规范文档时，必须同步更新本清单并在 [CHANGELOG.md](file:///Users/grandboy/Desktop/grandboy/gb-project-guidelines/4.%20FrontEnd/AI_RULES/CHANGELOG.md) 中记录版本演进。
