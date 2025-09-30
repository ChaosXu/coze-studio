# WorkflowPage 组件详解

## 概述

WorkflowPage 是工作流编辑页面的入口组件，负责渲染整个工作流编辑界面。它通过路由 `/work_flow?workflow_id=${workflowId}&space_id=${sId}` 访问，是用户编辑和管理工作流的核心页面。

## 组件实现

### 主要文件

- 入口文件: [frontend/packages/workflow/adapter/playground/src/page.tsx](../../frontend/packages/workflow/adapter/playground/src/page.tsx)
- 路由配置: [frontend/apps/coze-studio/src/routes/index.tsx](../../frontend/apps/coze-studio/src/routes/index.tsx)

### 核心组件结构

```tsx
export function WorkflowPage(): React.ReactNode {
  const workflowPlaygroundRef = useRef<WorkflowPlaygroundRef>(null);
  const {
    spaceId,
    workflowId,
    version,
    setVersion,
    from,
    optType,
    nodeId,
    executeId,
    subExecuteId,
  } = usePageParams();

  const [initOnce, setInitOnce] = useState(false);
  const { navigateBack } = useNavigateBack();

  /** 是否为只读模式，来源于流程探索模块 */
  const readonly = from === 'explore';

  if (!workflowId || !spaceId) {
    return null;
  }

  return (
    <>
      <WorkflowPlayground
        ref={workflowPlaygroundRef}
        sidebar={EmptySidebar}
        workflowId={workflowId}
        spaceId={spaceId}
        commitId={setVersion ? undefined : version}
        commitOptType={setVersion ? undefined : optType}
        readonly={readonly}
        executeId={executeId}
        subExecuteId={subExecuteId}
        onInit={_workflowState => {
          if (setVersion && version) {
            workflowPlaygroundRef.current?.resetToHistory({
              commitId: version,
              optType,
            });
          }

          // onInit可能会被调用多次，只需要执行一次
          if (!initOnce) {
            // 读取链接上的node_id参数，并滚动到对应的节点
            if (nodeId) {
              workflowPlaygroundRef.current?.scrollToNode(nodeId);
            }

            // 读取execute_id展示对应的执行结果
            if (executeId) {
              workflowPlaygroundRef.current?.showTestRunResult(
                executeId,
                subExecuteId,
              );
            }

            setInitOnce(true);
          }
        }}
        from={from}
        onBackClick={workflowState => {
          navigateBack(workflowState, 'exit');
        }}
        onPublish={workflowState => {
          navigateBack(workflowState, 'publish');
        }}
      />
    </>
  );
}
```

## 关键 Hooks

### usePageParams Hook

该 Hook 负责解析 URL 参数，获取工作流编辑所需的各项参数。

```typescript
interface SearchParams {
  workflow_id: string;           // 工作流ID
  space_id: string;              // 空间ID
  version?: string;              // 工作流版本，多人协作时有版本概念，设置后可以预览对应版本的流程
  set_version?: string;          // 是否恢复到目标版本，如果设置，则会自动将流程草稿设置为对应版本
  opt_type?: string;             // 对应版本的操作类型
  from?: WorkflowPlaygroundProps['from']; // 流程页面打开来源
  node_id?: string;              // 节点ID配置，会自动定位到对应节点
  execute_id?: string;           // 执行ID配置，会展示对应的执行结果
  sub_execute_id?: string;       // 子流程执行ID
}
```

### useNavigateBack Hook

负责处理页面返回逻辑，根据不同场景返回到合适的页面。

## 路由配置

在 [frontend/apps/coze-studio/src/routes/index.tsx](../../frontend/apps/coze-studio/src/routes/index.tsx) 中配置了工作流页面的路由：

```tsx
{
  path: 'work_flow',
  Component: WorkflowPage,
  loader: () => ({
    hasSider: false,
    requireAuth: true,
  }),
}
```

## 功能特性

1. **参数解析**: 从 URL 中提取工作流 ID、空间 ID 和其他可选参数
2. **历史版本支持**: 支持查看和恢复到特定的历史版本
3. **节点定位**: 可以通过 node_id 参数直接定位到特定节点
4. **执行结果显示**: 通过 execute_id 和 sub_execute_id 显示测试执行结果
5. **返回处理**: 智能处理返回逻辑，根据来源决定返回到哪个页面
6. **只读模式**: 支持从流程探索模块进入的只读模式

## 页面展示

当用户通过资源列表点击工作流资源时，会被重定向到 `/work_flow?workflow_id=${workflowId}&space_id=${sId}` 页面。该页面会显示完整的可视化工作流编辑器，允许用户查看、编辑和测试工作流。
