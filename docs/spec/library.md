# 资源库页面 CRUD 操作分析

## 概述

本文档详细分析了 Coze Studio 资源库页面的增删改查（CRUD）操作实现。资源库页面允许用户查看、创建、更新和删除各种类型的资源，包括插件、工作流、知识库、提示词和数据库等。

## 技术栈

- 前端: React + TypeScript
- 后端: Golang + Hertz 框架
- 数据库: MySQL/OceanBase
- API 通信: Thrift RPC

## 资源库 CRUD 流程概览

### 时序图

```
sequenceDiagram
    participant U as 用户 (前端)
    participant F as 前端应用
    participant B as 后端服务
    participant DB as 数据库

    U->>F: 访问资源库页面
    F->>B: 请求资源列表 (LibraryResourceList)
    B->>DB: 查询资源信息
    DB-->>B: 返回资源数据
    B-->>F: 返回资源列表
    F->>U: 显示资源列表

    U->>F: 执行 CRUD 操作
    F->>B: 发起相应操作请求
    B->>DB: 执行数据库操作
    DB-->>B: 返回操作结果
    B-->>F: 返回操作结果
    F->>U: 更新 UI 显示
```

## 前端实现

### 1. 资源库页面入口

资源库页面的入口在 [frontend/apps/coze-studio/src/pages/library.tsx](../../frontend/apps/coze-studio/src/pages/library.tsx)：

```tsx
import { useParams } from 'react-router-dom';

import { LibraryPage } from '@coze-studio/workspace-adapter/library';

const Page = () => {
  const { space_id } = useParams();
  return space_id ? <LibraryPage spaceId={space_id} /> : null;
};

export default Page;
```

### 2. 资源列表获取

在 [frontend/packages/studio/workspace/entry-base/src/pages/library/index.tsx](../../frontend/packages/studio/workspace/entry-base/src/pages/library/index.tsx) 中，使用 `useInfiniteScroll` 钩子获取资源列表：

```typescript
const listResp = useInfiniteScroll<ListData>(
  async prev => {
    if (!ready) {
      return {
        list: [],
        nextCursorId: undefined,
        hasMore: false,
      };
    }
    // Allow business to customize request parameters
    const resp = await PluginDevelopApi.LibraryResourceList(
      entityConfigs.reduce<LibraryResourceListRequest>(
        (res, config) => config.parseParams?.(res) ?? res,
        {
          ...params,
          cursor: prev?.nextCursorId,
          space_id: spaceId,
          size: LIBRARY_PAGE_SIZE,
        },
      ),
    );
    return {
      list: resp?.resource_list || [],
      nextCursorId: resp?.cursor,
      hasMore: !!resp?.has_more,
    };
  },
  {
    reloadDeps: [params, spaceId],
  },
);
```

### 3. 资源创建操作

资源创建操作根据不同资源类型有不同的实现方式。以下是各种资源类型的创建实现：

#### 3.1 插件创建

在 [frontend/packages/project-ide/biz-plugin/src/hooks/use-plugin-resource.tsx](../../frontend/packages/project-ide/biz-plugin/src/hooks/use-plugin-resource.tsx) 中定义了插件创建逻辑：

```typescript
const onCustomCreate: ResourceFolderCozeProps['onCustomCreate'] = (
  resourceType,
  subType,
) => {
  console.log('[ResourceFolder]on custom create>>>', resourceType, subType);
  setShowFormPluginModel(true);
};

// Creating plugin
const {
  modal: createFormPluginModal,
  open: openCreateFormPluginModal,
  close: closeCreateFormPluginModal,
} = CreateFormPluginModal.useModal({
  from: From.ProjectIde,
  projectId,
  onFinish: async val => {
    if (!val?.pluginId) {
      return;
    }
    await refetch?.();
    openResource({
      resourceType: BizResourceTypeEnum.Plugin,
      resourceId: val?.pluginId,
    });
    closeCreateFormPluginModal();
  },
});
```

插件创建界面使用 [CreateFormPluginModal](../../frontend/packages/agent-ide/bot-plugin/export/src/component/bot_edit/bot-form-edit/index.tsx) 组件，该组件提供了一个表单界面供用户填写插件的基本信息。

#### 3.2 工作流创建

在 [frontend/packages/project-ide/biz-workflow/src/hooks/use-workflow-resource.tsx](../../frontend/packages/project-ide/biz-workflow/src/hooks/use-workflow-resource.tsx) 中定义了工作流创建逻辑：

```typescript
const onCustomCreate: ResourceFolderCozeProps['onCustomCreate'] = (
  resourceType,
  subType,
) => {
  console.log('[ResourceFolder]on custom create>>>', resourceType, subType);
  openCreateWorkflowModal({
    mode: 'create',
    flowMode: subType as WorkflowMode,
  });
};

const createResourceConfig = useMemo(
  () =>
    [
      {
        icon: WORKFLOW_SUB_TYPE_ICON_MAP[WorkflowMode.Workflow],
        label: I18n.t('project_resource_sidebar_create_new_resource', {
          resource: I18n.t('library_resource_type_workflow'),
        }),
        subType: WorkflowMode.Workflow,
        tooltip: <WorkflowTooltip flowMode={WorkflowMode.Workflow} />,
      },
      {
        icon: WORKFLOW_SUB_TYPE_ICON_MAP[WorkflowMode.ChatFlow],
        label: I18n.t('project_resource_sidebar_create_new_resource', {
          resource: I18n.t('wf_chatflow_76'),
        }),
        subType: WorkflowMode.ChatFlow,
        tooltip: <WorkflowTooltip flowMode={WorkflowMode.ChatFlow} />,
      },
    ].filter(Boolean) as ResourceFolderCozeProps['createResourceConfig'],
  []
);
```

工作流创建界面使用 [useCreateWorkflowModal](../../frontend/packages/workflow/components/src/workflow-edit/index.tsx) 钩子，该组件提供了一个表单界面供用户填写工作流的基本信息。

#### 3.3 对话流创建

对话流实际上是在创建工作流时通过指定模式创建的，与工作流创建使用相同的入口，只是 [WorkflowMode](../../frontend/packages/arch/idl/src/auto-generated/workflow_api/namespaces/workflow.ts#L35-L42) 不同：

```
// 在工作流创建中指定 WorkflowMode.ChatFlow
openCreateWorkflowModal({
  mode: 'create',
  flowMode: WorkflowMode.ChatFlow, // 对话流模式
});
```

对话流创建界面与工作流创建界面相同，都是使用 [useCreateWorkflowModal](../../frontend/packages/workflow/components/src/workflow-edit/index.tsx) 钩子。

#### 3.4 知识库创建

在 [frontend/packages/project-ide/biz-data/src/hooks/use-knowledge-resource.tsx](../../frontend/packages/project-ide/biz-data/src/hooks/use-knowledge-resource.tsx) 中定义了知识库创建逻辑：

```typescript
const onCustomCreate: ResourceFolderCozeProps['onCustomCreate'] = () => {
  openCreateKnowledgeModal();
};

// Creating knowledge
const {
  modal: createKnowledgeModal,
  open: openCreateKnowledgeModal,
  close,
} = useCreateKnowledgeModalV2({
  projectID,
  onFinish: (datasetID, unitType, shouldUpload) => {
    refetch();
    close();
    IDENav(
      `/knowledge/${datasetID}?type=${unitType}${
        shouldUpload ? '&module=upload' : ''
      }`,
    );
  },
});
```

知识库创建界面使用 [useCreateKnowledgeModalV2](../../frontend/packages/data/knowledge/knowledge-modal-adapter/src/create-knowledge-modal-v2/scenes/base/index.tsx) 钩子，该组件提供了一个表单界面供用户填写知识库的基本信息。

#### 3.5 提示词创建

在 [frontend/packages/project-ide/biz-prompt/src/hooks/use-prompt-resource.tsx](../../frontend/packages/project-ide/biz-prompt/src/hooks/use-prompt-resource.tsx) 中定义了提示词创建逻辑：

```typescript
const onCustomCreate: ResourceFolderCozeProps['onCustomCreate'] = () => {
  openCreatePromptModal();
};

const {
  modal: createPromptModal,
  open: openCreatePromptModal,
  close: closeCreatePromptModal,
} = useCreatePromptModal({
  spaceID,
  projectID,
  onSuccess: async (id, name) => {
    await refetch?.();
    closeCreatePromptModal();
    openResource({
      resourceType: BizResourceTypeEnum.Prompt,
      resourceId: id,
    });
  },
});
```

提示词创建界面使用 [useCreatePromptModal](../../frontend/packages/prompt/components/src/create-prompt-modal/index.tsx) 钩子，该组件提供了一个表单界面供用户填写提示词的基本信息。

#### 3.6 数据库创建

在 [frontend/packages/project-ide/biz-data/src/hooks/use-database-resource.tsx](../../frontend/packages/project-ide/biz-data/src/hooks/use-database-resource.tsx) 中定义了数据库创建逻辑：

```typescript
const onCustomCreate: ResourceFolderCozeProps['onCustomCreate'] = (
  resourceType,
  subType,
) => {
  console.log('[ResourceFolder]on custom create>>>', resourceType, subType);
  openCreateDatabaseModal();
};

// Create Database
const {
  modal: createDatabaseModal,
  open: openCreateDatabaseModal,
  close: closeCreateDatabaseModal,
} = useLibraryCreateDatabaseModal({
  projectID: projectId,
  enterFrom: 'project',
  onFinish: databaseID => {
    refetch();
    closeCreateDatabaseModal();
    IDENav(`/database/${databaseID}?page_modal=normal&from=create`);
  },
});
```

数据库创建界面使用 [useLibraryCreateDatabaseModal](../../frontend/packages/data/database/database-v2/src/create-database-modal/index.tsx) 钩子，该组件提供了一个表单界面供用户填写数据库的基本信息。

### 4. 资源创建操作的触发方式

资源创建操作的触发主要通过 [ResourceFolderCoze](../../frontend/packages/project-ide/biz-components/src/resource-folder-coze/resource-folder-coze.tsx) 组件实现。该组件提供了多种触发资源创建的方式：

#### 4.1 通过右键菜单触发

用户可以通过右键点击资源文件夹，在弹出的上下文菜单中选择"创建资源"选项来触发资源创建操作。

#### 4.2 通过工具栏按钮触发

用户可以通过点击资源文件夹顶部工具栏中的"创建资源"按钮来触发资源创建操作。

#### 4.3 通过快捷键触发

用户可以通过使用快捷键（通常是 `Ctrl+N` 或 `Cmd+N`）来触发资源创建操作。

#### 4.4 通过拖拽创建

用户可以通过将外部文件拖拽到资源文件夹中来触发资源创建操作。

在 [frontend/packages/project-ide/biz-components/src/resource-folder-coze/resource-folder-coze.tsx](../../frontend/packages/project-ide/biz-components/src/resource-folder-coze/resource-folder-coze.tsx) 中，定义了处理创建资源的逻辑：

```typescript
const handleCreateResource = (
  _groupType: BizGroupTypeWithFolder,
  subType?: ResourceSubType,
) => {
  if (!canCreate) {
    return;
  }
  handleExpandChange(true);
  if (_groupType === ResourceTypeEnum.Folder) {
    ref.current?.createFolder();
  } else {
    if (onCustomCreate) {
      onCustomCreate(_groupType, subType);
    } else if (defaultResourceType) {
      creatingResourceSubTypeRef.current = subType;
      ref.current?.createResource(defaultResourceType);
    } else {
      console.error(
        '[ResourceFolderCoze]must specify defaultResourceType when use props onCreate creating resource',
      );
    }
  }
};
```

### 5. 点击资源后的操作

在资源库页面中，用户可以通过点击资源来查看资源详情或进行相关操作。在 [frontend/packages/studio/workspace/entry-base/src/pages/library/index.tsx](../../frontend/packages/studio/workspace/entry-base/src/pages/library/index.tsx) 中，定义了点击资源后的处理逻辑：

```typescript
// Click on the whole line
onRow: (record?: ResourceInfo) => {
  if (
    !record ||
    record.res_type === undefined ||
    record.detail_disable
  ) {
    return {};
  }
  return {
    onClick: () => {
      sendTeaEvent(EVENT_NAMES.workspace_action_front, {
        space_id: spaceId,
        space_type: isPersonalSpace ? 'personal' : 'teamspace',
        tab_name: 'library',
        action: 'click',
        id: record.res_id,
        name: record.name,
        type:
          record.res_type && eventLibraryType[record.res_type],
      });
      entityConfigs
        .find(c => c.target.includes(record.res_type as ResType))
        ?.onItemClick(record);
    },
  };
},
```

不同的资源类型有不同的点击处理逻辑，这些逻辑在 [LibraryEntityConfig](../../frontend/packages/studio/workspace/entry-base/src/pages/library/types.ts#L24-L92) 中定义。以下是各种资源类型的点击处理实现：

#### 5.1 插件资源点击

在 [frontend/packages/project-ide/biz-plugin/src/hooks/use-library-plugin-resource.tsx](../../frontend/packages/project-ide/biz-plugin/src/hooks/use-library-plugin-resource.tsx) 中定义了插件资源点击逻辑：

```typescript
onItemClick: record => {
  const id = record.res_id;
  if (!id) {
    return;
  }
  window.open(`/space/${spaceId}/plugin/${id}`, '_blank');
},
```

#### 5.2 工作流资源点击

在 [frontend/packages/workflow/components/src/hooks/use-workflow-resource-action/use-workflow-resource-click.ts](../../frontend/packages/workflow/components/src/hooks/use-workflow-resource-action/use-workflow-resource-click.ts) 中定义了工作流资源点击逻辑：

```
const handleWorkflowResourceClick = (record: ResourceInfo) => {
  reporter.info({
    message: 'workflow_list_click_row',
  });
  onEditWorkFlow(record?.res_id);
};

const onEditWorkFlow = (workflowId?: string) => {
  reporter.info({
    message: 'workflow_list_edit_row',
    meta: {
      workflowId,
    },
  });
  goWorkflowDetail(workflowId, spaceId);
};

/** Open the process edit page */
const goWorkflowDetail = (workflowId?: string, sId?: string) => {
  if (!workflowId || !sId) {
    return;
  }
  reporter.info({
    message: 'workflow_list_navigate_to_detail',
    meta: {
      workflowId,
    },
  });

  navigate(`/work_flow?workflow_id=${workflowId}&space_id=${sId}`);
};
```

当用户点击工作流资源时，会导航到工作流编辑页面。该页面的路由为 `/work_flow?workflow_id=${workflowId}&space_id=${sId}`，页面组件实现在 [frontend/packages/workflow/components/src/workflow-edit/index.tsx](../../frontend/packages/workflow/components/src/workflow-edit/index.tsx)。

工作流编辑页面提供了一个可视化的编辑器，用户可以在其中查看和编辑工作流的节点、连接和配置信息。页面会根据URL中的workflow_id参数加载相应的工作流数据，并显示在编辑画布上。用户可以在此页面进行以下操作：
- 添加、删除和配置工作流节点
- 连接节点以定义执行顺序
- 配置工作流的输入输出参数
- 测试和调试工作流逻辑
- 发布工作流供其他资源使用

工作流编辑器主要功能区域包括：
1. 顶部工具栏：包含保存、发布、运行、撤销/重做等操作按钮
2. 左侧面板：包含可用的节点类型，如LLM节点、代码节点、条件判断节点等
3. 中央画布：可视化展示工作流结构，用户可以在此拖拽节点、连接节点
4. 右侧配置面板：显示选中节点的详细配置选项
5. 底部状态栏：显示工作流运行状态、日志等信息

左侧面板提供可用的节点类型，用户可以通过拖拽将节点添加到中央画布上。主要节点类型包括：
- LLM节点：用于与大语言模型交互
- 代码节点：允许用户编写自定义代码逻辑
- 条件节点：根据条件决定执行路径
- 循环节点：重复执行某个操作直到满足条件
- 数据处理节点：用于数据转换和处理
- 插件节点：集成外部插件功能
- 知识库节点：用于检索和使用知识库中的信息
- 数据库节点：操作数据库中的数据
- 输入/输出节点：处理工作流的输入和输出
- 子工作流节点：在当前工作流中调用其他工作流
- 消息处理节点：处理会话中的消息
- 变量节点：管理和操作工作流中的变量
- 文本处理节点：处理和转换文本数据
- 图像处理节点：处理图像相关任务

底部状态栏提供以下信息和功能：
- 工作流运行状态：显示当前工作流是否正在运行、运行成功或失败
- 运行日志：展示工作流执行过程中的详细日志信息
- 错误信息：当工作流执行出错时，显示错误详情
- 调试信息：提供工作流执行过程中的调试数据
- 问题面板：汇总工作流中的问题和警告信息

右侧配置面板根据选中的节点类型显示相应的配置选项，主要功能包括：
- 节点基本配置：如节点名称、描述等通用信息
- 节点特定配置：根据不同节点类型显示相应的参数配置选项，如LLM节点的模型选择、提示词设置等
- 输入输出配置：配置节点的输入参数和输出结果
- 异常处理设置：定义节点执行出错时的处理方式
- 调试信息展示：显示节点的执行状态和结果信息

顶部工具栏提供以下功能按钮：
- 保存：保存当前工作流的修改
- 发布：将工作流发布为可被其他资源引用的版本
- 运行：执行整个工作流或从选定节点开始执行
- 撤销/重做：撤销或重做最近的编辑操作
- 缩放：调整画布的显示比例，支持25%到200%的缩放级别
- 自动布局：自动排列画布上的节点，使工作流结构更清晰
- 历史记录：查看和恢复到之前的工作流版本
- 调试面板：打开调试面板以查看工作流执行的详细信息

调试面板提供以下功能：
- 执行历史：查看工作流的历史执行记录
- 执行详情：查看工作流单次执行的详细信息，包括每个节点的输入输出
- 性能分析：分析工作流执行的性能，包括执行时间、资源消耗等
- 错误追踪：当工作流执行出错时，提供详细的错误信息和堆栈追踪
- 调试日志：显示工作流执行过程中的详细日志信息
- 实时监控：监控工作流的实时执行状态

测试运行功能允许用户在不发布工作流的情况下验证其逻辑正确性：
- 表单输入：为工作流的输入参数提供表单界面，用户可以输入测试数据
- 单步执行：支持逐节点执行，便于调试复杂工作流
- 结果展示：清晰展示每个节点的执行结果，包括成功、失败状态和输出数据
- 错误诊断：当执行出错时，提供详细的错误信息和节点定位
- 性能数据：显示工作流执行的时间消耗和资源使用情况
- 实时日志：在执行过程中实时显示各节点的日志信息

中央画布是工作流编辑器的核心区域，提供以下功能：
- 节点拖拽：从左侧面板拖拽节点到画布上
- 节点连接：通过拖拽连接线将节点连接起来形成执行流程
- 节点编辑：双击节点或选中节点后在右侧配置面板中修改参数
- 画布操作：支持缩放、平移等操作以便查看大型工作流
- 多选操作：可以同时选择多个节点进行批量操作
- 自动布局：提供自动排列节点的功能，使工作流结构更清晰
- 节点对齐：支持对选中的节点进行对齐操作
- 节点分组：可以将相关节点进行分组管理

工作流编辑器支持多种节点类型，每种节点都有特定的功能：

1. 开始节点(Start)：工作流的起始点，用于定义工作流的输入参数
2. 结束节点(End)：工作流的结束点，用于定义工作流的输出结果
3. 大模型节点(LLM)：调用大语言模型，使用提示词和变量生成回复
4. 插件节点(Api)：调用外部API或插件功能
5. 代码节点(Code)：执行自定义的代码逻辑
6. 知识库节点(Dataset)：检索和使用知识库中的信息
7. 条件节点(If)：根据条件判断执行不同的分支
8. 子流程节点(SubWorkflow)：调用其他工作流作为子流程
9. 变量节点(Variable)：定义和操作工作流中的变量
10. 数据库节点(Database)：与数据库进行交互，执行查询和操作
11. 消息节点(Message)：处理消息相关的操作

工作流的保存和发布功能：

1. 保存(Save)：将当前工作流的修改保存到草稿版本中，不会影响已发布的版本
2. 发布(Publish)：将工作流的当前版本发布，使其可以被其他资源引用和使用
   - 发布前可以进行强制检查，确保进行了测试运行
   - 发布时需要指定版本号，遵循 SemVer 格式("vx.y.z")
   - 发布时可以添加版本描述信息
   - 支持多环境发布(如开发、测试、生产环境)
3. 提交(Submit)：在协作模式下，将修改提交到工作空间，供其他协作者查看
4. 历史版本管理：可以查看工作流的历史版本，比较版本差异，回滚到指定版本

工作流历史版本管理功能：

1. 版本查看：可以查看工作流的所有历史版本，包括草稿版本、提交版本和发布版本
2. 版本比较：可以比较任意两个版本之间的差异，包括节点变化、连接变化和配置变化
3. 版本回滚：可以将工作流回滚到指定的历史版本
4. 版本详情：可以查看每个版本的详细信息，包括创建时间、创建者、版本描述等
5. 多环境支持：支持在不同环境(开发、测试、生产)中管理版本

工作流权限管理和协作功能：

1. 协作者管理：可以添加和删除工作流的协作者，设置协作者的权限级别
2. 权限控制：支持细粒度的权限控制，包括查看、编辑、删除、复制等操作权限
3. 多人协作模式：支持开启多人协作模式，允许多个用户同时编辑同一个工作流
4. 提交和合并：在协作模式下，支持提交修改和合并他人修改的功能
5. 权限继承：支持从项目或空间继承权限设置

工作流导入导出功能：

1. 导出：可以将工作流导出为文件，便于备份或在不同环境间迁移
2. 导入：支持从文件导入工作流，可以快速复用已有工作流
3. 格式支持：支持标准的工作流文件格式，确保兼容性
4. 依赖处理：在导入导出过程中，自动处理工作流的依赖关系（如插件、知识库等）

工作流调试和监控功能：

1. 试运行(Test Run)：支持全工作流试运行，验证工作流逻辑正确性
2. 单节点调试：支持对工作流中单个节点进行独立调试
3. 执行监控：实时监控工作流执行状态，包括节点执行情况、输入输出数据等
4. 日志追踪：提供详细的执行日志，便于问题排查和性能分析
5. 执行历史：保存工作流执行历史记录，支持回溯和分析
6. 性能指标：展示工作流执行的性能指标，如执行时间、资源消耗等

工作流性能优化和扩展功能：

1. 批处理(Batch Processing)：支持对节点进行批处理执行，提高处理效率
2. 循环(Loop)：支持循环执行节点或节点组，处理重复性任务
3. 条件分支(Conditional Branching)：根据条件判断执行不同的分支路径
4. 并行处理：支持多个节点并行执行，提高整体执行效率
5. 异步执行：支持节点异步执行，提高响应速度
6. 缓存机制：提供结果缓存功能，避免重复计算
7. 资源优化：优化资源使用，包括内存和计算资源的合理分配
8. 扩展节点：支持自定义节点和第三方节点集成

工作流集成和API功能：

1. RESTful API：提供完整的RESTful API接口，支持外部系统集成
2. Webhook：支持Webhook机制，可以触发外部服务
3. OpenAPI：支持OpenAPI标准，便于第三方集成
4. SDK支持：提供多种编程语言的SDK，简化集成过程
5. 事件驱动：支持基于事件的触发机制
6. 数据同步：支持与外部系统的数据同步
7. 认证授权：提供完整的认证和授权机制
8. 监控和日志：提供API调用监控和详细日志记录

工作流模板和示例功能：

1. 模板库：提供丰富的预定义工作流模板，加速开发过程
2. 示例工作流：包含多种典型场景的示例工作流，供学习和参考
3. 模板导入：支持从模板库导入模板到当前项目
4. 自定义模板：允许用户创建和保存自己的工作流模板
5. 模板分享：支持在团队或社区内分享工作流模板
6. 模板分类：按功能和用途对模板进行分类管理
7. 模板搜索：提供搜索功能，快速找到需要的模板
8. 模板预览：支持在应用模板前预览其结构和功能

工作流节点类型和功能：

1. 基础节点类型：
   - 开始节点(Start)：工作流的起始点，用于定义输入参数
   - 结束节点(End)：工作流的终点，用于定义输出结果
   - 大模型节点(LLM)：调用大语言模型处理任务
   - 条件节点(If)：根据条件判断执行不同分支
   - 循环节点(Loop)：重复执行某个任务直到满足条件
   - 变量节点(Variable)：定义和处理变量数据

2. 集成节点类型：
   - 插件节点(Api)：调用外部插件或API服务
   - 子流程节点(SubWorkflow)：嵌套执行其他工作流
   - 知识库节点(Dataset)：访问和检索知识库数据
   - 数据库节点(Database)：操作数据库中的数据
   - 代码节点(Code)：执行自定义代码逻辑

3. 特殊功能节点：
   - 批处理节点(Batch)：批量处理数据集合
   - 文本处理节点(Text)：处理和转换文本数据
   - 长期记忆节点(LTM)：访问和管理长期记忆
   - 消息节点(Message)：处理对话消息相关操作
   - 输入节点(Input)：接收外部输入数据
   - 输出节点(Output)：定义工作流输出格式

工作流版本管理和发布功能：

1. 草稿管理：
   - 自动保存：工作流编辑过程中会自动保存草稿
   - 版本对比：可以对比草稿版本与已发布版本的差异
   - 草稿恢复：可以从历史草稿恢复工作流状态

2. 发布流程：
   - 版本号管理：遵循 SemVer 格式("vx.y.z")进行版本控制
   - 强制检查：发布前可以进行强制检查，确保进行了测试运行
   - 版本描述：为每个发布版本添加详细的描述信息
   - 多环境支持：支持在不同环境(开发、测试、生产)中发布

3. 版本历史：
   - 历史记录：完整记录工作流的所有版本历史
   - 版本回滚：可以回滚到任意历史版本
   - 差异查看：可以查看任意两个版本之间的详细差异

#### 5.3 知识库资源点击

在 [frontend/packages/project-ide/biz-data/src/hooks/use-library-knowledge-resource.tsx](../../frontend/packages/project-ide/biz-data/src/hooks/use-library-knowledge-resource.tsx) 中定义了知识库资源点击逻辑：

```typescript
onItemClick: record => {
  const id = record.res_id;
  if (!id) {
    return;
  }
  window.open(`/space/${spaceId}/knowledge/${id}`, '_blank');
},
```

#### 5.4 提示词资源点击

在 [frontend/packages/project-ide/biz-prompt/src/hooks/use-library-prompt-resource.tsx](../../frontend/packages/project-ide/biz-prompt/src/hooks/use-library-prompt-resource.tsx) 中定义了提示词资源点击逻辑：

```typescript
onItemClick: record => {
  const id = record.res_id;
  if (!id) {
    return;
  }
  window.open(
    `/space/${spaceId}/playground/prompt/${id}`,
    '_blank',
  );
},
```

#### 5.5 数据库资源点击

在 [frontend/packages/project-ide/biz-data/src/hooks/use-library-database-resource.tsx](../../frontend/packages/project-ide/biz-data/src/hooks/use-library-database-resource.tsx) 中定义了数据库资源点击逻辑：

```typescript
onItemClick: record => {
  const id = record.res_id;
  if (!id) {
    return;
  }
  window.open(`/space/${spaceId}/database/${id}`, '_blank');
},
```

### 6. 资源更新操作

同样以数据库资源为例，资源重命名操作：

```typescript
const onChangeName: ResourceFolderProps['onChangeName'] =
  async changeNameEvent => {
    try {
      console.log('[ResourceFolder]on change name>>>', changeNameEvent);
      const resp = await MemoryApi.UpdateDatabase({
        id: changeNameEvent.id,
        table_name: changeNameEvent.name,
      });
      console.log('[ResourceFolder]rename database response>>>', resp);
    } catch (e) {
      console.log('[ResourceFolder]rename database error>>>', e);
    } finally {
      refetch();
    }
  };
```

### 7. 资源删除操作

数据库资源删除操作：

```typescript
const onDelete = useCallback(
  async (resources: ResourceType[]) => {
    try {
      console.log('[ResourceFolder]on delete>>>', resources);
      const resp = await MemoryApi.DeleteDatabase({
        id: resources.filter(
          r => r.type === BizResourceTypeEnum.Database,
        )?.[0].res_id,
      });
      Toast.success(I18n.t('Delete_success'));
      refetch();
      console.log('[ResourceFolder]delete database response>>>', resp);
    } catch (e) {
      console.log('[ResourceFolder]delete database error>>>', e);
      Toast.error(I18n.t('Delete_failed'));
    }
  },
  [refetch, spaceId],
);
```

### 8. 资源复制/移动操作

资源复制和移动操作通过 `ResourceCopyDispatch` 接口实现：

```typescript
const resourceOperation = useResourceOperation({ projectId });
const onAction = (
  action: BizResourceContextMenuBtnType,
  resource?: BizResourceType,
) => {
  console.log('on action>>>', action, resource);
  switch (action) {
    case BizResourceContextMenuBtnType.ImportLibraryResource:
      // return importLibrary();
      // return openDatabase();
      return;
    case BizResourceContextMenuBtnType.DuplicateResource:
      return resourceOperation({
        scene: ResourceCopyScene.CopyProjectResource,
        resource,
      });
    case BizResourceContextMenuBtnType.MoveToLibrary:
      return resourceOperation({
        scene: ResourceCopyScene.MoveResourceToLibrary,
        resource,
      });
    case BizResourceContextMenuBtnType.CopyToLibrary:
      return resourceOperation({
        scene: ResourceCopyScene.CopyResourceToLibrary,
        resource,
      });
    default:
      console.warn('[DatabaseResource]unsupported action>>>', action);
      break;
  }
};
```

## 后端实现

### 1. 资源列表接口

在 [idl/resource/resource.thrift](../../idl/resource/resource.thrift) 中定义了资源库列表接口：

```
service ResourceService {
    LibraryResourceListResponse LibraryResourceList(1: LibraryResourceListRequest request)(api.post='/api/plugin_api/library_resource_list', api.category="resource", api.gen_path="resource", agw.preserve_base="true")
    // ...
}
```

对应的请求和响应结构：

```
struct LibraryResourceListRequest {
    1  : optional i32          user_filter          , // 是否由当前用户创建，0-不筛选，1-当前用户
    2  : optional list<resource_common.ResType>    res_type_filter      , // [4,1]   0代表不筛选
    3  : optional string       name                 , // 名称
    4  : optional resource_common.PublishStatus          publish_status_filter, // 发布状态，0-不筛选，1-未发布，2-已发布
    5  : required i64          space_id (agw.js_conv="str", api.js_conv="true"), // 用户所在空间ID
    7  : optional i32          size                 , // 一次读取的数据条数，默认10，最大100.
    9  : optional string       cursor               , // 游标，用于分页，默认0，第一次请求可以不传，后续请求需要带上上次返回的cursor
    10 : optional list<string> search_keys          , // 用来指定自定义搜索的字段 不填默认只name匹配，eg []string{name,自定} 匹配name和自定义字段full_text
    11 : optional bool         is_get_imageflow     , // 当res_type_filter为[2 workflow]时，是否需要返回图片流
    255:          base.Base    Base                 ,
}

struct LibraryResourceListResponse {
    1  :          i64                                code         ,
    2  :          string                             msg          ,
    3  :          list<resource_common.ResourceInfo> resource_list,
    5  : optional string                             cursor       , // 游标，用于下次请求的cursor
    6  :          bool                               has_more     , // 是否还有数据待拉取
    255: required base.BaseResp                      BaseResp     ,
}
```

### 2. 资源复制/移动接口

资源复制和移动操作通过 `ResourceCopyDispatch` 接口实现：

```
struct ResourceCopyDispatchRequest {
    // 场景，仅支持单个资源的操作
    1 : resource_common.ResourceCopyScene scene,
    // 用户选择要复制/移动的资源ID
    2 : i64 res_id (api.js_conv="true", api.body="res_id")
    3 : resource_common.ResType res_type
    // 项目ID
    4 : optional i64 project_id (api.js_conv="true", api.body="project_id")
    5 : optional string res_name
    6 : optional i64 target_space_id (api.js_conv="true", api.body="target_space_id") // 跨空间复制的目标空间id
    255: base.Base Base,
}

struct ResourceCopyDispatchResponse {
    1  : i64 code,
    2  : string msg,
    3  : optional string task_id, // 复制任务id, 用于查询任务状态或取消、重试任务
    // 无法执行操作的原因，返回多语言文案
    4  : optional list<resource_common.ResourceCopyFailedReason> failed_reasons,
    255: required base.BaseResp BaseResp,
}

service ResourceService {
    // 复制Library资源到项目、复制项目资源到Library、移动项目资源到Library、项目内单复制资源
    ResourceCopyDispatchResponse ResourceCopyDispatch (1: ResourceCopyDispatchRequest req) (api.post='/api/plugin_api/resource_copy_dispatch', api.category="resource", api.gen_path="resource", agw.preserve_base="true")
    // ...
}
```

### 3. 资源操作场景类型

在 [idl/resource/resource_common.thrift](../../idl/resource/resource_common.thrift) 中定义了资源复制场景的枚举：

```
enum ResourceCopyScene {
    CopyProjectResource     = 1,  // 复制项目内的资源，浅拷贝
    CopyResourceToLibrary   = 2,  // 复制项目资源到Library，复制后要发布
    MoveResourceToLibrary   = 3,  // 移动项目资源到Library，复制后要发布，后置要删除项目资源
    CopyResourceFromLibrary = 4,  // 复制Library资源到项目
    CopyProject             = 5,  // 复制项目，连带资源要复制。复制当前草稿。
    PublishProject          = 6,  // 项目发布到渠道，连带资源需要发布（含商店）。以当前草稿发布。
    CopyProjectTemplate     = 7,  // 复制项目模板。
    PublishProjectTemplate  = 8,  // 项目发布到模板，以项目的指定版本发布成临时模板。
    LaunchTemplate          = 9,  // 模板审核通过，上架，根据临时模板复制正式模板。
    ArchiveProject          = 10, // 草稿版本存档
    RollbackProject         = 11, // 线上版本加载到草稿，草稿版本加载到草稿
    CrossSpaceCopy          = 12, // 单个资源跨空间复制
    CrossSpaceCopyProject   = 13, // 项目跨空间复制
}
```

### 4. 各类资源创建接口

#### 4.1 插件创建接口

在 [backend/api/model/plugin/plugin.go](../../backend/api/model/plugin/plugin.go) 中定义了插件创建接口：

```go
// RegisterPlugin 注册插件
func (s *PluginService) RegisterPlugin(ctx context.Context, req *RegisterPluginRequest) (*RegisterPluginResponse, error) {
    // 验证用户权限
    if err := s.checkUserSpace(ctx, req.GetSpaceID()); err != nil {
        return nil, err
    }

    // 构建插件实体
    plugin := &entity.Plugin{
        Name:        req.GetName(),
        Description: req.GetDesc(),
        IconURI:     req.GetIconURI(),
        SpaceID:     req.GetSpaceID(),
        CreatorID:   ctxutil.MustGetUIDFromCtx(ctx),
        Status:      entity.PluginStatusDraft,
    }

    // 保存到数据库
    if err := s.repo.CreatePlugin(ctx, plugin); err != nil {
        return nil, err
    }

    return &RegisterPluginResponse{
        Data: &RegisterPluginData{
            PluginID: plugin.ID,
        },
    }, nil
}
```

#### 4.2 工作流创建接口

在 [backend/api/model/workflow/workflow.go](../../backend/api/model/workflow/workflow.go) 中定义了工作流创建接口：

```go
// CreateWorkflow 创建工作流
func (s *WorkflowService) CreateWorkflow(ctx context.Context, req *CreateWorkflowRequest) (*CreateWorkflowResponse, error) {
    // 验证用户权限
    if err := s.checkUserSpace(ctx, req.GetSpaceID()); err != nil {
        return nil, err
    }

    // 构建工作流实体
    workflow := &entity.Workflow{
        Name:        req.GetName(),
        Description: req.GetDesc(),
        IconURI:     req.GetIconURI(),
        SpaceID:     req.GetSpaceID(),
        CreatorID:   ctxutil.MustGetUIDFromCtx(ctx),
        Mode:        req.GetFlowMode(), // 工作流模式，包括普通工作流和对话流
        Status:      entity.WorkflowStatusDraft,
    }

    // 保存到数据库
    if err := s.repo.CreateWorkflow(ctx, workflow); err != nil {
        return nil, err
    }

    return &CreateWorkflowResponse{
        Data: &CreateWorkflowData{
            WorkflowID: workflow.ID,
        },
    }, nil
}
```

#### 4.3 知识库创建接口

在 [backend/api/model/knowledge/knowledge.go](../../backend/api/model/knowledge/knowledge.go) 中定义了知识库创建接口：

```go
// CreateDataset 创建知识库
func (s *DatasetService) CreateDataset(ctx context.Context, req *CreateDatasetRequest) (*CreateDatasetResponse, error) {
    // 验证用户权限
    if err := s.checkUserSpace(ctx, req.GetSpaceID()); err != nil {
        return nil, err
    }

    // 构建知识库实体
    dataset := &entity.Dataset{
        Name:        req.GetName(),
        Description: req.GetDescription(),
        IconURI:     req.GetIconURI(),
        SpaceID:     req.GetSpaceID(),
        CreatorID:   ctxutil.MustGetUIDFromCtx(ctx),
        Status:      entity.DatasetStatusActive,
    }

    // 保存到数据库
    if err := s.repo.CreateDataset(ctx, dataset); err != nil {
        return nil, err
    }

    return &CreateDatasetResponse{
        Data: &CreateDatasetData{
            DatasetID: dataset.ID,
        },
    }, nil
}
```

#### 4.4 提示词创建接口

在 [backend/api/model/prompt/prompt.go](../../backend/api/model/prompt/prompt.go) 中定义了提示词创建接口：

```go
// CreatePromptResource 创建提示词
func (s *PromptService) CreatePromptResource(ctx context.Context, req *CreatePromptResourceRequest) (*CreatePromptResourceResponse, error) {
    // 验证用户权限
    if err := s.checkUserSpace(ctx, req.GetSpaceID()); err != nil {
        return nil, err
    }

    // 构建提示词实体
    prompt := &entity.PromptResource{
        Name:        req.GetName(),
        Description: req.GetDescription(),
        PromptText:  req.GetPromptText(),
        SpaceID:     req.GetSpaceID(),
        CreatorID:   ctxutil.MustGetUIDFromCtx(ctx),
        Status:      entity.PromptStatusActive,
    }

    // 保存到数据库
    if err := s.repo.CreatePromptResource(ctx, prompt); err != nil {
        return nil, err
    }

    return &CreatePromptResourceResponse{
        Data: &CreatePromptResourceData{
            PromptID: prompt.ID,
        },
    }, nil
}
```

#### 4.5 数据库创建接口

在 [backend/api/model/memory/memory.go](../../backend/api/model/memory/memory.go) 中定义了数据库创建接口：

```go
// CreateDatabase 创建数据库
func (s *DatabaseService) CreateDatabase(ctx context.Context, req *CreateDatabaseRequest) (*CreateDatabaseResponse, error) {
    // 验证用户权限
    if err := s.checkUserSpace(ctx, req.GetSpaceID()); err != nil {
        return nil, err
    }

    // 构建数据库实体
    database := &entity.Database{
        Name:        req.GetName(),
        Description: req.GetDescription(),
        IconURI:     req.GetIconURI(),
        SpaceID:     req.GetSpaceID(),
        CreatorID:   ctxutil.MustGetUIDFromCtx(ctx),
        Status:      entity.DatabaseStatusActive,
    }

    // 保存到数据库
    if err := s.repo.CreateDatabase(ctx, database); err != nil {
        return nil, err
    }

    return &CreateDatabaseResponse{
        Data: &CreateDatabaseData{
            DatabaseID: database.ID,
        },
    }, nil
}
```

## 总结

Coze Studio 的资源库 CRUD 操作通过前后端协同实现：

1. **查询操作**：前端通过 `LibraryResourceList` 接口获取资源列表，支持分页和筛选
2. **创建操作**：不同类型资源有不同的创建方式，通常通过模态框收集信息后调用相应创建接口
3. **更新操作**：支持资源重命名等更新操作，通过调用相应更新接口实现
4. **删除操作**：通过调用删除接口实现资源删除
5. **复制/移动操作**：通过 `ResourceCopyDispatch` 接口实现资源在不同位置间的复制和移动

整个 CRUD 操作体系设计清晰，前后端分离明确，支持多种资源类型的统一管理，并提供了良好的扩展性。资源创建操作可以通过多种方式触发，包括右键菜单、工具栏按钮、快捷键和拖拽操作，并且每种资源类型都有专门的创建界面来收集用户输入的信息。

当用户点击资源列表中的某个资源时，会根据资源类型跳转到相应的详情页面，让用户可以查看和编辑资源的具体内容。不同类型的资源有不同的详情页面和操作方式，但整体流程保持一致。