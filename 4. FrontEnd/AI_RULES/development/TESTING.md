# 质量保证与测试规范 (TESTING)

规范编号：DEV-003  
版本：1.0.0  
适用范围：强制执行  

---

## 1. 测试体系与分层模型

本项目对 Web 管理后台的核心业务逻辑、数据流 Hook 与公共组件执行自动化测试保障：

```
        / \
       /   \      Playwright 端到端关键链路测试 (登录、核心表单流)
      /-----\
     /       \    Component 组件与页面集成测试 (@testing-library/react)
    /---------\
   /           \  Unit 单元测试 (Hooks 数据流 & 工具函数)
  /-------------\
```

- **Hooks 测试 (Hook Tests)**：使用 `renderHook` 验证基于 TanStack Query 的数据加载、缓存重用与异常处理。
- **组件与页面测试 (Component Tests)**：验证组件在 Loading、Empty、Error 与 Success 四态下的渲染表现、交互事件触发与表单校验。
- **网络模拟 (MSW)**：使用 Mock Service Worker 拦截网络请求，保证测试脱离真实后端且稳定可重放。

---

## 2. 源码与测试目录镜像规范

测试文件紧贴被测试源文件放置或统一放置在 `tests/` 目录下：

```
src/domains/devices/hooks/useDevices.ts
└── src/domains/devices/hooks/__tests__/useDevices.test.ts

src/domains/devices/pages/DeviceListPage.tsx
└── src/domains/devices/pages/__tests__/DeviceListPage.test.tsx
```

---

## 3. 标准测试范式 (`Vitest` & `@testing-library/react`)

### 3.1 测试 Query Hook 范式

```typescript
import { renderHook, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { describe, it, expect, vi } from 'vitest';
import { useDevicesQuery } from '../useDevices';
import * as deviceApi from '../../api/device.api';

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
    },
  });
  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  );
};

describe('useDevicesQuery', () => {
  it('正确拉取设备列表并返回数据', async () => {
    const mockData = { list: [{ id: '1', name: 'Smart Bulb A' }], total: 1 };
    vi.spyOn(deviceApi, 'fetchDeviceList').mockResolvedValueOnce(mockData);

    const { result } = renderHook(() => useDevicesQuery({ page: 1, pageSize: 10 }), {
      wrapper: createWrapper(),
    });

    expect(result.current.isLoading).toBe(true);

    await waitFor(() => expect(result.current.isSuccess).toBe(true));
    expect(result.current.data).toEqual(mockData);
  });
});
```

---

## 4. 本地提交与 CI 质量门禁套件

任何代码提交或任务交付前，必须在工程根目录下执行并通过以下三道质量门禁：

```bash
# 1. 代码格式化与风格规范检查
pnpm lint

# 2. 严格 TypeScript 编译与类型检查
pnpm exec tsc --noEmit

# 3. 运行全量自动化测试套件
pnpm test
```
