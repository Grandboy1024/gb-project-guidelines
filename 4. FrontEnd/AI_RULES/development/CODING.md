# React & TypeScript 编码与防御规范 (CODING)

规范编号：DEV-001  
版本：1.0.0  
适用范围：强制执行  

---

## 1. Modern TypeScript 5.x 强类型规范

1. **严格类型安全**：
   - `tsconfig.json` 开启 `"strict": true`。
   - **绝对禁止使用 `any`**；对于暂时未知的动态数据，使用 `unknown` 并配合类型守卫（Type Guard）或 Zod 校验。
   - 严禁使用 `@ts-ignore` 或 `as any` 绕过编译报错。
2. **类型与契约推导 (Zod)**：
   - 统一使用 Zod 声明 Schema，并通过 `z.infer<typeof schema>` 自动推导 TypeScript 类型：
     ```typescript
     import { z } from 'zod';

     export const deviceSchema = z.object({
       id: z.string().uuid(),
       name: z.string().min(1).max(50),
       model: z.string(),
       isOnline: z.boolean(),
       createdAt: z.string().datetime(),
     });

     export type Device = z.infer<typeof deviceSchema>;
     ```

---

## 2. TanStack Query v5 最佳实践

1. **QueryKey 数组集中化管理**：
   - QueryKey 必须使用结构化数组，保持依赖项（如当前页码、过滤关键字）与缓存强绑定：
     ```typescript
     export const deviceKeys = {
       all: ['devices'] as const,
       lists: () => [...deviceKeys.all, 'list'] as const,
       list: (params: DeviceQueryParams) => [...deviceKeys.lists(), params] as const,
       details: () => [...deviceKeys.all, 'detail'] as const,
       detail: (id: string) => [...deviceKeys.details(), id] as const,
     };
     ```
2. **数据变更与缓存主动失效**：
   - 执行 `useMutation` 提交修改后，必须主动触发受影响 query 的失效刷新，严禁前端手工拼接状态：
     ```typescript
     const queryClient = useQueryClient();
     const mutation = useMutation({
       mutationFn: updateDevice,
       onSuccess: (_, variables) => {
         queryClient.invalidateQueries({ queryKey: deviceKeys.lists() });
         queryClient.invalidateQueries({ queryKey: deviceKeys.detail(variables.id) });
         message.success('设备更新成功');
       },
     });
     ```

---

## 3. UI 设计系统与 Ant Design 5.x 规范

1. **Design Tokens 驱动**：
   - 界面间距、圆角与色彩一律通过 Ant Design ConfigProvider 统一配置的主题 Tokens（如 `token.colorPrimary`, `token.borderRadius`）或 Tailwind CSS 标准类名读取。
   - 严禁手写奇数魔法像素（如 `padding: 13px`）或硬编码十六进制色值（如 `#1890ff`）。
2. **国际化与多语言 (`react-i18next`)**：
   - 界面所有静态文本、弹窗标题、校验错误信息一律通过 `t('device.title')` 读取，严禁中英文硬编码在 JSX 中。

---

## 4. UI 边界条件与极端场景防御

### 4.1 异步数据四态闭环 (Loading / Empty / Error / Success)

依赖远端 API 的页面或组件，**必须完整处理四态**，严禁出现空白无反馈页面：

```tsx
export const DeviceList = () => {
  const { data, isLoading, isError, refetch } = useDevicesQuery(params);

  if (isLoading) {
    return <Skeleton active paragraph={{ rows: 6 }} />;
  }

  if (isError) {
    return (
      <Result
        status="error"
        title="加载设备列表失败"
        subTitle="网络开小差了，请检查网络后重试"
        extra={<Button type="primary" onClick={() => refetch()}>重新加载</Button>}
      />
    );
  }

  if (!data?.list?.length) {
    return (
      <Empty
        description="暂无设备数据"
        image={Empty.PRESENTED_IMAGE_SIMPLE}
      >
        <Button type="primary" onClick={handleOpenAddModal}>立即添加设备</Button>
      </Empty>
    );
  }

  return <Table dataSource={data.list} rowKey="id" columns={columns} />;
};
```

### 4.2 长文本排版与表格防溢出

1. **表格单元格长文本防溢出**：
   - 表格中可能超长的文本列（如设备名称、SN 码、Mac 地址）必须配置 `ellipsis: true`，并外层包裹 Tooltip：
     ```tsx
     {
       title: '设备名称',
       dataIndex: 'name',
       ellipsis: true,
       render: (text: string) => <Tooltip title={text}>{text}</Tooltip>,
     }
     ```
2. **单行溢出截断**：
   - 卡片标题与描述使用 Tailwind `truncate` 或 CSS `text-overflow: ellipsis; white-space: nowrap; overflow: hidden;`。

### 4.3 交互防重与输入防抖

1. **按钮防重复提交**：
   - 提交表单或发起异步请求时，按钮必须绑定加载态并禁用：
     ```tsx
     <Button type="primary" loading={mutation.isPending} disabled={mutation.isPending} onClick={handleSubmit}>
       提交保存
     </Button>
     ```
2. **搜索输入防抖 (Debounce)**：
   - 搜索输入框实时筛选必须使用 `useDebounce`（300~500ms），防范过度请求拖垮服务端。
3. **列表渲染 key 铁律**：
   - 动态列表渲染必须使用唯一实体主键：`key={item.id}`，**严禁使用数组下标 `key={index}`**。
