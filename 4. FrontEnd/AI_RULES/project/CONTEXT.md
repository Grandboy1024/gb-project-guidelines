# 业务背景与工程全景 (CONTEXT)

规范编号：PROJECT-001  
版本：1.0.0  
适用范围：强制执行  

---

## 1. 产品概述与设计目标

本项目是基于 **React 18/19**、**TypeScript 5.x** 与 **Vite** 构建的高性能现代化 B 端 Web 运营管理后台。

系统的核心定位是面向企业运营人员、硬件支持工程师与管理员的综合控制中枢，核心设计目标包括：
1. **高效敏捷的运营管理体验**：基于 Ant Design 5.x 规范，提供统一规范的表格查询、复杂表单交互、批量操作与数据可视化大盘。
2. **高鲁棒性与零白屏防御**：推行深度防御性编程，对网络中断、慢网加载、数据为空、长文本溢出及表单并发重复提交进行全局防护与四态闭环。
3. **清晰的模块化与低心智负担**：采用领域就近聚合 (Colocation) 架构，彻底分离服务端状态 (TanStack Query) 与客户端轻量状态 (Zustand)。

---

## 2. 核心业务功能模块全景

所有业务领域代码统一组织在 `src/domains/` 目录下，各模块完全自闭环：

```
src/domains/
├── auth/           # 运营人员认证与权限系统 (登录、登出、Token 自动刷新)
├── users/          # 用户中心 (注册用户列表、家庭架构、设备归属检索)
├── products/       # 产品型号中心 (智能灯物模型属性定义、功能集与硬件品类)
├── orders/         # 交易与订单中心 (商城订单流水、交易状态流转、退款审批)
├── devices/        # 设备台账与监控中枢 (在线设备大盘、MQTT 实时状态查看、远程调试控制)
└── ota/            # 固件升级与版本发布 (固件验签上传、灰度/全量升级任务下发、进度看板)
```

---

## 3. 基础设施与跨模块共享服务

```
src/
├── app/
│   ├── layouts/           # 页面框架布局 (MainLayout 包含侧边栏导航，AuthLayout 居中卡片)
│   ├── providers/         # 全局 Provider 组合 (QueryProvider, ThemeProvider)
│   └── router/            # React Router 路由注册与 AuthGuard 路由守卫
├── core/                  # 与具体业务无关的纯系统底层运行支撑
│   ├── config/env.ts      # 集中环境变量读取与 Zod 校验
│   ├── http/client.ts     # Axios 封装 (统一错误解包、Token 拦截、取消令牌)
│   ├── logging/logger.ts  # 统一日志接口
│   └── errors/            # ErrorBoundary 渲染异常边界兜底
└── shared/                # 跨 Domain 通用的无业务属性 UI 组件与工具库
    ├── components/        # Button, Modal, Table, Pagination, Loading, EmptyState
    ├── hooks/             # useDebounce, useMediaQuery
    └── utils/             # date.ts, format.ts
```

---

## 4. 关键工程质量优先级

1. **状态分离第一铁律**：严禁将服务端 API 数据写入 Zustand；服务端数据统一由 TanStack Query 托管。
2. **零白屏与四态闭环**：异步数据加载必须完整闭环 Loading、Empty、Error（带重试按钮）与 Success。
3. **严格强类型保障**：全面开启 TypeScript 严格模式，严禁使用 `any` 逃避类型检查。
