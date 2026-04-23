# AGENTS.md — SoybeanAdmin

> AI 编码代理项目上下文指引。Vue3 + Vite8 + TypeScript6 + Pinia3 + NaiveUI + UnoCSS + Elegant Router。

## 环境与命令

- **Node.js** >= 20.19.0 | **pnpm** >= 10.5.0 | **严禁使用 npm / yarn**
- `pnpm dev`（端口 9527）| `pnpm build` | `pnpm lint` | `pnpm fmt` | `pnpm typecheck`
- `pnpm gen-route` 手动生成路由 | `pnpm commit` 生成规范提交信息
- 路径别名：`@` → `./src`，`~` → `./`

## 自动化文件路由（Elegant Router）

- 在 `src/views/` 下新建 `.vue` → 路由自动生成；删除 → 路由自动移除
- `src/router/elegant/` 为**自动生成目录，禁止手动编辑**
- `<template>` 中**必须只有一个根元素**（`<Transition>` 要求）
- 自定义路由 meta 在 `src/router/routes/index.ts` 的 `customRoutes` 数组中配置
- `onRouteMetaGen`（`build/plugins/router.ts`）自动为每个路由设置 `title: routeName` 和 `i18nKey: route.${routeName}`

**路由 Meta 常用属性**：`title` | `i18nKey` | `roles` | `keepAlive` | `constant` | `icon` | `localIcon` | `order` | `hideInMenu` | `href` | `activeMenu` | `multiTab` | `fixedIndexInTab` | `query`

**布局**：`base`（`src/layouts/base-layout/`）| `blank`（`src/layouts/blank-layout/`）

**权限模式**（`.env` `VITE_AUTH_ROUTE_MODE`）：`static`（默认，`roles` 字段控制）| `dynamic`（后端 API）

## 新增页面（完整流程）

### 步骤 1：创建页面文件

```vue
<script setup lang="ts">
import { computed } from 'vue';
import { useAppStore } from '@/store/modules/app';
import { $t } from '@/locales';
import SubModule from './modules/sub-module.vue';

// keepAlive 缓存时必须定义 name（与路由名一致）
defineOptions({ name: 'MyPageName' });

const appStore = useAppStore();
const gap = computed(() => (appStore.isMobile ? 0 : 16));
</script>

<template>
  <NSpace vertical :size="16">
    <NCard :bordered="false" class="card-wrapper">
      <SubModule />
    </NCard>
  </NSpace>
</template>

<style scoped></style>
```

**关键规则**：
- 页面子组件统一放 `modules/` 子目录（不用 `components/`）
- NaiveUI 组件、`SvgIcon`、`CountTo` 通过 `unplugin-vue-components` **自动导入，无需 import**
- NaiveUI 使用 PascalCase：`<NButton>` `<NCard>` `<NGrid>` `<NGi>` `<NDataTable>` 等
- 全局方法：`window.$message` `window.$dialog` `window.$notification` `window.$loadingBar`

### 步骤 2：添加国际化翻译（必须）

同时修改 `src/locales/langs/zh-cn.ts` 和 `en-us.ts`：

```typescript
// 1. 路由标题 → route 对象（key 为路由名）
route: { 'my-new-page': '我的新页面' }

// 2. 页面内容 → page 对象
page: { myNewPage: { title: '页面标题', someLabel: '某标签' } }
```

**必须更新 i18n Schema 类型**（`src/typings/app.d.ts` → `namespace I18n` → `Schema` → `page`）：
```typescript
page: { myNewPage: { title: string; someLabel: string } }
```
> ⚠️ 不更新 Schema 类型会导致 TypeScript 编译错误，这是最容易遗漏的步骤。

### 步骤 3：路由 meta（可选）

默认自动生成 `title` + `i18nKey` + `constant`。需自定义 `icon`、`order`、`roles` 等时，在 `customRoutes` 中配置。

## CRUD 表格页面

项目封装了完整的表格 Hooks（`src/hooks/common/table.ts`）：

```vue
<script setup lang="ts">
import { $t } from '@/locales';
import { useNaivePaginatedTable, useTableOperate, defaultTransform } from '@/hooks/common/table';
import { fetchGetXxxList } from '@/service/api';

defineOptions({ name: 'XxxManage' });

const columns = () => [
  { type: 'selection', key: 'selection' },
  { title: $t('page.xxx.name'), key: 'name', minWidth: 120 },
  { title: $t('page.xxx.status'), key: 'status', width: 100 },
  { title: $t('common.action'), key: 'action', width: 200, render: (row) => { /* 操作按钮 */ } }
];

const { data, columns: tableColumns, loading, pagination, mobilePagination, getData, getDataByPage } =
  useNaivePaginatedTable({
    apiFn: fetchGetXxxList,
    apiParams: { current: 1, size: 10 },
    columns,
    transformer: defaultTransform
  });

const { drawerVisible, operateType, editingData, handleAdd, handleEdit, checkedRowKeys, onDeleted, onBatchDeleted } =
  useTableOperate(data, 'id', getData);
</script>
```

## ECharts 图表页面

```vue
<script setup lang="ts">
import { useEcharts } from '@/hooks/common/echarts';
import type { ECOption } from '@/hooks/common/echarts';

const { domRef, updateOptions } = useEcharts(() => ({
  tooltip: { trigger: 'axis' },
  xAxis: { type: 'category', data: ['Mon', 'Tue', 'Wed'] },
  yAxis: { type: 'value' },
  series: [{ type: 'line', data: [120, 200, 150] }]
} as ECOption));

// 异步更新：await updateOptions(opts => { opts.series[0].data = newData; return opts; });
</script>

<template>
  <NCard :bordered="false" class="card-wrapper">
    <div ref="domRef" class="h-360px"></div>
  </NCard>
</template>
```

支持图表类型：Bar、Line、Pie、Scatter、PictorialBar、Radar、Gauge。

## 新增 API 接口

1. 在 `src/typings/api/` 定义类型（`declare namespace Api` 结构）
2. 在 `src/service/api/` 定义函数，在 `index.ts` 统一导出
3. 命名约定：`fetch` + 动词 + 名词

```typescript
import { request } from '../request';

export function fetchGetUserList(params?: Api.SystemManage.UserSearchParams) {
  return request<Api.Common.PaginatingQueryRecord<Api.SystemManage.User>>({
    url: '/systemManage/getUserList', method: 'get', params
  });
}
```

分页响应格式（`src/typings/api/common.d.ts`）：`{ records: T[], current, size, total }`

## 新增 Store 模块

1. 在 `src/store/modules/` 下创建目录
2. 使用 **Setup Store** 写法，Store ID 在 `src/enum/index.ts` 的 `SetupStoreId` 枚举注册
3. **直接 import 使用**，无需在 `store/index.ts` 导出

```typescript
import { ref } from 'vue';
import { defineStore } from 'pinia';
import { SetupStoreId } from '@/enum';

export const useMyStore = defineStore(SetupStoreId.MyModule, () => {
  const someState = ref('');
  function someAction() { /* ... */ }
  return { someState, someAction };
});
```

## 常用 Hooks 速查

| Hook | 来源 | 用途 |
|------|------|------|
| `useEcharts` | `@/hooks/common/echarts` | ECharts 图表 |
| `useNaivePaginatedTable` | `@/hooks/common/table` | 分页表格 |
| `useTableOperate` | `@/hooks/common/table` | 表格 CRUD 操作 |
| `useFormRules` | `@/hooks/common/form` | 表单校验 |
| `useRouterPush` | `@/hooks/common/router` | 路由跳转 |
| `useAuth` | `@/hooks/business/auth` | 权限判断 |
| `useBoolean` | `@sa/hooks` | 布尔状态（`{ bool, setTrue, setFalse, toggle }`） |
| `useLoading` | `@sa/hooks` | 加载状态（`{ loading, startLoading, endLoading }`） |

`@vueuse/core` 常用：`useBreakpoints` | `useElementSize` | `useTitle` | `useEventListener` | `createReusableTemplate`

## 样式约定

- UnoCSS 原子类：`px-16px` `text-16px` `h-360px` `flex` `justify-between` 等
- 卡片容器：`card-wrapper` 类名
- 响应式网格：`<NGrid cols="s:1 m:2 l:4" responsive="screen">`
- 组件样式：`<style scoped></style>`（即使为空也保留）

## 注意事项

1. **严禁 npm / yarn**，必须 pnpm
2. **`src/router/elegant/` 自动生成**，禁止手动修改
3. `<template>` 只能有**一个根元素**
4. NaiveUI 组件**自动导入**，无需 import
5. 全局方法挂载在 `window`：`$message` `$dialog` `$notification` `$loadingBar`
6. 使用 `localStg`（`@/utils/storage`）封装，**不要直接用 `localStorage`**
7. **新增页面必须更新 i18n**：`zh-cn.ts` + `en-us.ts` + `app.d.ts` Schema 类型
8. **Store 直接 import**，`store/index.ts` 仅安装 Pinia；新增 Store 先注册 `SetupStoreId` 枚举
9. 全局组件放 `src/components/{common,advanced,custom}/`，页面组件放 `modules/`
10. 图标：Iconify（`icon` 属性如 `mdi:menu`）或本地 SVG（`src/assets/svg-icon/`，`localIcon` 引用）
11. 请求封装：`createFlatRequest` 返回 `{ data, error }`（`src/service/request/`），成功 code 由 `VITE_SERVICE_SUCCESS_CODE`（默认 `0000`）控制
12. 主题配置：`src/theme/settings.ts`（色彩/布局）| `src/theme/vars.ts`（CSS 变量）| `uno.config.ts`（UnoCSS）
