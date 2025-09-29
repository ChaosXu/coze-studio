# Coze Studio 前端架构分析

## 项目结构

```
frontend/
├── apps/                          # 应用目录
│   └── coze-studio/               # 主应用
│       ├── src/                   # 源代码目录
│       │   ├── app.tsx            # 应用入口文件
│       │   ├── layout.tsx         # 全局布局组件
│       │   ├── index.tsx          # 项目入口文件
│       │   ├── pages/             # 页面组件目录
│       │   │   ├── develop.tsx    # 开发页面
│       │   │   ├── explore.tsx    # 探索页面
│       │   │   ├── library.tsx    # 资源库页面
│       │   │   ├── plugin/        # 插件相关页面
│       │   │   │   ├── page.tsx   # 插件主页面
│       │   │   │   └── tool/      # 工具相关页面
│       │   │   │       └── page.tsx  # 插件工具页面
│       │   │   └── redirect.tsx   # 重定向页面
│       │   └── routes/            # 路由配置
│       │       ├── index.tsx      # 路由主配置
│       │       └── async-components.tsx  # 异步加载组件
│       └── ...
├── packages/                      # 共享包目录
│   ├── agent-ide/                 # 智能体IDE相关组件
│   ├── arch/                      # 架构相关包
│   ├── foundation/                # 基础组件和适配器
│   ├── studio/                    # 工作室相关组件
│   └── workflow/                  # 工作流相关组件
└── ...
```

## 技术栈

- React 18.x
- TypeScript
- React Router v6
- Vite/Rsbuild 构建工具
- TailwindCSS 样式框架
- Zustand 状态管理
- i18n 国际化

## 核心架构设计

### 1. 微前端架构

Coze Studio 采用微前端架构，将不同功能模块拆分为独立的包：

- `@coze-agent-ide/*` - 智能体开发相关组件
- `@coze-community/*` - 社区探索相关组件
- `@coze-foundation/*` - 基础设施和布局组件
- `@coze-project-ide/*` - 项目开发相关组件
- `@coze-studio/*` - 工作室核心组件
- `@coze-workflow/*` - 工作流相关组件

### 2. 路由系统

前端使用 React Router v6 实现路由管理，采用嵌套路由结构。

#### 路由配置示例

```tsx
// frontend/apps/coze-studio/src/routes/index.tsx
export const router: ReturnType<typeof createBrowserRouter> =
  createBrowserRouter([
    {
      path: '/',
      Component: Layout,
      errorElement: <GlobalError />,
      children: [
        {
          index: true,
          element: <Navigate to="/space" replace />,
        },
        {
          path: 'sign',
          Component: LoginPage,
          errorElement: <GlobalError />,
          loader: () => ({
            hasSider: false,
            requireAuth: false,
          }),
        },
        {
          path: 'space',
          Component: SpaceLayout,
          loader: () => ({
            hasSider: true,
            requireAuth: true,
            subMenu: spaceSubMenu,
            menuKey: BaseEnum.Space,
          }),
          children: [
            {
              path: ':space_id',
              Component: SpaceIdLayout,
              children: [
                {
                  index: true,
                  element: <Navigate to="develop" replace />,
                },
                {
                  path: 'develop',
                  Component: Develop,
                  loader: () => ({
                    subMenuKey: SpaceSubModuleEnum.DEVELOP,
                  }),
                },
                {
                  path: 'bot/:bot_id',
                  Component: AgentIDELayout,
                  children: [
                    {
                      index: true,
                      Component: AgentIDE,
                    },
                    {
                      path: 'publish',
                      children: [
                        {
                          index: true,
                          Component: AgentPublishPage,
                          loader: () => ({
                            hasSider: false,
                            requireBotEditorInit: false,
                            pageName: 'publish',
                          }),
                        },
                      ],
                    },
                  ],
                  loader: () => ({
                    hasSider: false,
                    showMobileTips: true,
                    requireBotEditorInit: true,
                    pageName: 'bot',
                  }),
                },
              ],
            },
          ],
        },
      ],
    },
  ]);
```

### 3. 页面组件结构

#### 应用入口

```tsx
// frontend/apps/coze-studio/src/app.tsx
import { RouterProvider } from 'react-router-dom';
import { Suspense } from 'react';

import { Spin } from '@coze-arch/coze-design';

import { router } from './routes';

export function App() {
  return (
    <Suspense
      fallback={
        <div className="w-full h-full flex items-center justify-center">
          <Spin spinning style={{ height: '100%', width: '100%' }} />
        </div>
      }
    >
      <RouterProvider router={router} fallbackElement={<div>loading...</div>} />
    </Suspense>
  );
}
```

#### 全局布局

```tsx
// frontend/apps/coze-studio/src/layout.tsx
import { GlobalLayout, useAppInit } from '@coze-foundation/global-adapter';

export const Layout = () => {
  useAppInit();

  return <GlobalLayout />;
};
```

#### 页面组件示例 - 开发页面

```tsx
// frontend/apps/coze-studio/src/pages/develop.tsx
import { useParams } from 'react-router-dom';

import { Develop } from '@coze-studio/workspace-adapter/develop';

const Page = () => {
  const { space_id } = useParams();
  return space_id ? <Develop spaceId={space_id} /> : null;
};

export default Page;
```

#### 页面组件示例 - 插件页面

```tsx
// frontend/apps/coze-studio/src/pages/plugin/page.tsx
import { useParams } from 'react-router-dom';
import { useEffect } from 'react';

import { Plugin } from '@coze-studio/workspace-base';
import { usePluginStoreInstance } from '@coze-studio/bot-plugin-store';

const Page = () => {
  const { plugin_id, space_id } = useParams();
  const pluginStore = usePluginStoreInstance();
  if (!plugin_id || !space_id) {
    throw Error('[plugin render error]: need plugin id and space id');
  }
  useEffect(() => {
    pluginStore?.getState().init();
  }, []);
  return <Plugin />;
};

export default Page;
```

#### 页面组件示例 - 插件工具页面

```tsx
// frontend/apps/coze-studio/src/pages/plugin/tool/page.tsx
import { useParams } from 'react-router-dom';
import { useEffect } from 'react';

import { Tool } from '@coze-studio/workspace-base';
import { usePluginStoreInstance } from '@coze-studio/bot-plugin-store';
const Page = () => {
  const { plugin_id, space_id, tool_id } = useParams();
  const pluginStore = usePluginStoreInstance();
  if (!plugin_id || !space_id || !tool_id) {
    throw Error('[plugin render error]: need plugin id and space id');
  }
  useEffect(() => {
    pluginStore?.getState().init();
  }, []);
  return <Tool toolID={tool_id} />;
};

export default Page;
```

### 4. 异步组件加载

```tsx
// frontend/apps/coze-studio/src/routes/async-components.tsx
import { lazy } from 'react';

// 登录页面
export const LoginPage = lazy(() =>
  import('@coze-foundation/account-ui-adapter').then(res => ({
    default: res.LoginPage,
  })),
);

// 工作区布局组件
export const SpaceLayout = lazy(() =>
  import('@coze-foundation/space-ui-adapter').then(exps => ({
    default: exps.SpaceLayout,
  })),
);

// 工作区特定ID布局组件
export const SpaceIdLayout = lazy(() =>
  import('@coze-foundation/space-ui-base').then(exps => ({
    default: exps.SpaceIdLayout,
  })),
);

// 项目开发页面
export const Develop = lazy(() => import('../pages/develop'));

// 智能体IDE布局组件
export const AgentIDELayout = lazy(
  () => import('@coze-agent-ide/layout-adapter'),
);

// 智能体IDE页面
export const AgentIDE = lazy(() =>
  import('@coze-agent-ide/entry-adapter').then(res => ({
    default: res.BotEditor,
  })),
);
```

## 状态管理

使用 Zustand 进行状态管理，主要在以下包中：

- `@coze-studio/bot-plugin-store` - 插件状态管理
- `@coze-studio/bot-detail-store` - 智能体详情状态管理
- `@coze-arch/bot-studio-store` - 工作室状态管理

## 国际化

使用 `@coze-arch/i18n` 包进行国际化支持，支持中英文切换。

## 样式系统

使用 TailwindCSS 作为主要样式框架，并结合 Less 进行自定义样式开发。

## 构建系统

使用 Rsbuild 作为构建工具，配置文件位于 `rsbuild.config.ts`。

## 总结

Coze Studio 前端采用了现代化的 React 技术栈，通过微前端架构将功能模块拆分，提高了代码的可维护性和可扩展性。路由系统采用嵌套结构，页面组件通过异步加载提高性能。整体架构清晰，便于团队协作开发和后期维护。