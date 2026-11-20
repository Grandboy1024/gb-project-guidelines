```jsx

project/
├── .github/
│   ├── workflows/
│   │   └── ci.yml                       # 自动化检查、测试、构建
│   └── pull_request_template.md
│
├── public/                               # 不参与模块打包的静态资源
│   ├── favicon.ico
│   └── robots.txt
│
├── src/
│   ├── main.tsx                          # 应用启动入口
│   ├── App.tsx                           # 应用根组件
│   │
│   ├── app/                              # 应用级组装与配置
│   │   ├── router/
│   │   │   ├── index.tsx                 # 路由注册
│   │   │   ├── paths.ts                  # 路由路径常量
│   │   │   └── guards.tsx                # 登录、权限等路由守卫
│   │   ├── providers/
│   │   │   ├── QueryProvider.tsx         # TanStack Query 配置
│   │   │   ├── ThemeProvider.tsx          # 主题配置
│   │   │   └── AppProviders.tsx          # 全局 Provider 组合
│   │   ├── layouts/
│   │   │   ├── MainLayout.tsx            # 主应用布局
│   │   │   ├── AuthLayout.tsx            # 登录注册布局
│   │   │   └── components/
│   │   └── config/
│   │       └── navigation.ts             # 导航菜单配置
│   │
│   ├── core/                             # 底层运行支撑
│   │   ├── config/
│   │   │   └── env.ts                    # 环境变量读取与校验
│   │   ├── http/
│   │   │   ├── client.ts                 # HTTP 客户端实例
│   │   │   ├── interceptors.ts           # 请求、响应拦截
│   │   │   └── errors.ts                 # 网络错误处理
│   │   ├── logging/
│   │   │   └── logger.ts                 # 统一日志接口
│   │   └── errors/
│   │       └── ErrorBoundary.tsx         # React 渲染异常边界
│   │
│   ├── shared/                           # 跨业务复用能力
│   │   ├── components/
│   │   │   ├── Button/
│   │   │   ├── Modal/
│   │   │   ├── Table/
│   │   │   ├── Pagination/
│   │   │   ├── Loading/
│   │   │   └── EmptyState/
│   │   ├── hooks/
│   │   │   ├── useDebounce.ts
│   │   │   └── useMediaQuery.ts
│   │   ├── utils/
│   │   │   ├── date.ts
│   │   │   └── format.ts
│   │   ├── types/
│   │   │   └── common.ts
│   │   ├── constants/
│   │   │   └── index.ts
│   │   └── styles/
│   │       ├── variables.css
│   │       └── global.css
│   │
│   ├── services/                         # 通用技术服务
│   │   ├── storage/
│   │   │   └── local-storage.ts
│   │   ├── i18n/
│   │   │   ├── index.ts
│   │   │   └── locales/
│   │   │       ├── zh-CN.json
│   │   │       └── en-US.json
│   │   └── monitoring/
│   │       └── index.ts                  # 错误监控接入
│   │
│   ├── domains/                          # 按业务领域组织功能
│   │   ├── auth/
│   │   │   ├── api/
│   │   │   │   └── auth.api.ts           # 登录、刷新 Token 等
│   │   │   ├── components/
│   │   │   │   └── LoginForm.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useAuth.ts
│   │   │   ├── pages/
│   │   │   │   ├── LoginPage.tsx
│   │   │   │   └── RegisterPage.tsx
│   │   │   ├── schemas/
│   │   │   │   └── auth.schema.ts
│   │   │   ├── stores/
│   │   │   │   └── auth.store.ts
│   │   │   ├── types/
│   │   │   │   └── auth.types.ts
│   │   │   └── index.ts                  # 对外公开接口
│   │   │
│   │   ├── users/
│   │   │   ├── api/
│   │   │   │   └── users.api.ts
│   │   │   ├── components/
│   │   │   │   └── UserForm.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useUsers.ts
│   │   │   ├── pages/
│   │   │   │   ├── UserListPage.tsx
│   │   │   │   └── UserDetailPage.tsx
│   │   │   ├── schemas/
│   │   │   ├── types/
│   │   │   └── index.ts
│   │   │
│   │   ├── products/
│   │   │   ├── api/
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   ├── pages/
│   │   │   ├── schemas/
│   │   │   ├── types/
│   │   │   └── index.ts
│   │   │
│   │   └── orders/
│   │       ├── api/
│   │       ├── components/
│   │       ├── hooks/
│   │       ├── pages/
│   │       ├── schemas/
│   │       ├── types/
│   │       └── index.ts
│   │
│   ├── assets/                           # 需要被构建工具处理的资源
│   │   ├── images/
│   │   ├── icons/
│   │   └── fonts/
│   │
│   └── tests/                            # 跨业务测试及公共测试工具
│       ├── setup.ts
│       ├── mocks/
│       ├── fixtures/
│       └── e2e/
│
├── .env.example                          # 环境变量模板
├── .env.development                      # 本地开发配置（不提交敏感值）
├── .env.production                       # 生产环境非敏感配置
├── .editorconfig
├── .gitignore
├── .prettierrc.json
├── eslint.config.js
├── index.html
├── package.json
├── pnpm-lock.yaml
├── tsconfig.json
├── vite.config.ts
├── vitest.config.ts
├── playwright.config.ts
├── Dockerfile
├── docker-compose.yml
├── nginx.conf
├── Makefile
└── README.md

```