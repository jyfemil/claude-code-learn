# 第 5 章：工具系统设计

## 5.1 工具接口契约

### 5.1.1 Tool 类型定义

```typescript
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  // 核心方法
  call(args: z.infer<Input>, context: ToolUseContext, ...): Promise<ToolResult<Output>>
  description(input: z.infer<Input>, options): Promise<string>

  // 输入/输出
  readonly inputSchema: Input
  readonly inputJSONSchema?: ToolInputJSONSchema
  outputSchema?: z.ZodType<unknown>

  // 能力判断（关键设计）
  isEnabled(): boolean
  isConcurrencySafe(input: z.infer<Input>): boolean
  isReadOnly(input: z.infer<Input>): boolean
  isDestructive?(input: z.infer<Input>): boolean

  // 权限与验证
  validateInput?(input, context): Promise<ValidationResult>
  checkPermissions(input, context): Promise<PermissionResult>

  // UI 渲染
  renderToolUseMessage(input, options): React.ReactNode
  renderToolResultMessage(content, progress, options): React.ReactNode

  // ... 更多方法
}
```

### 5.1.2 buildTool 工厂模式

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?: unknown) => false,
  isReadOnly: (_input?: unknown) => false,
  isDestructive: (_input?: unknown) => false,
  checkPermissions: async (input, _ctx) => ({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: (_input?: unknown) => '',
  userFacingName: (_input?: unknown) => '',
}

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return { ...TOOL_DEFAULTS, userFacingName: () => def.name, ...def } as BuiltTool<D>
}
```

**设计亮点**：
- 工厂模式填充默认值，调用者无需处理 `?.() ?? default`
- fail-closed 原则：`isConcurrencySafe` 默认为 false（保守）、`isReadOnly` 默认为 false（假设写入）
- 所有工具通过 `buildTool()` 创建，确保接口一致性

## 5.2 工具注册机制

### 5.2.1 条件加载（Dead Code Elimination）

```typescript
// 基于 feature() 的条件导入
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null

// 基于环境变量的条件加载
const REPLTool = process.env.USER_TYPE === 'ant'
  ? require('./tools/REPLTool/REPLTool.js').REPLTool
  : null

// 运行时函数延迟加载（打破循环依赖）
const getPowerShellTool = () => {
  if (!isPowerShellToolEnabled()) return null
  return require('./tools/PowerShellTool/PowerShellTool.js').PowerShellTool
}
```

**设计亮点**：
- 编译时条件编译（`feature()` 在 bun build 时求值）
- 运行时延迟加载（解决循环依赖）
- 环境变量驱动（`USER_TYPE` 区分内部版本）

### 5.2.2 getAllBaseTools 工具池组装

```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    TaskOutputTool,
    BashTool,
    // 条件包含
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    ...(isTodoV2Enabled() ? [TaskCreateTool, TaskGetTool, TaskUpdateTool, TaskListTool] : []),
    ...(process.env.USER_TYPE === 'ant' ? [ConfigTool, TungstenTool] : []),
    // ... 更多工具
  ]
}
```

## 5.3 权限系统

### 5.3.1 ToolPermissionContext

```typescript
export type ToolPermissionContext = {
  mode: PermissionMode  // 'default' | 'auto' | 'bypass'
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource   // 白名单
  alwaysDenyRules: ToolPermissionRulesBySource    // 黑名单
  alwaysAskRules: ToolPermissionRulesBySource     // 询问规则
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  shouldAvoidPermissionPrompts?: boolean  // 后台代理不显示 UI
  awaitAutomatedChecksBeforeDialog?: boolean  // 协调器工作线程
}
```

### 5.3.2 工具过滤

```typescript
export function filterToolsByDenyRules(tools, permissionContext) {
  return tools.filter(tool => !getDenyRuleForTool(permissionContext, tool))
}

export function getTools(permissionContext) {
  // 1. 获取基础工具
  const tools = getAllBaseTools()

  // 2. 过滤被拒绝的工具
  let allowedTools = filterToolsByDenyRules(tools, permissionContext)

  // 3. REPL 模式下隐藏原始工具
  if (isReplModeEnabled()) {
    allowedTools = allowedTools.filter(tool => !REPL_ONLY_TOOLS.has(tool.name))
  }

  // 4. 检查 isEnabled()
  const isEnabled = allowedTools.map(_ => _.isEnabled())
  return allowedTools.filter((_, i) => isEnabled[i])
}
```

### 5.3.3 assembleToolPool 完整工具池

```typescript
export function assembleToolPool(permissionContext, mcpTools): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)

  // 按名称排序，内置工具保持为连续前缀（用于 prompt cache）
  const byName = (a, b) => a.name.localeCompare(b.name)
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name'
  )
}
```

**设计亮点**：
- 内置工具保持为连续前缀 → 缓存命中优化
- MCP 工具追加在后面，避免与内置工具混排导致缓存失效

## 5.4 工具执行流程

### 5.4.1 工具调用接口

```typescript
call(
  args: z.infer<Input>,
  context: ToolUseContext,
  canUseTool: CanUseToolFn,
  parentMessage: AssistantMessage,
  onProgress?: ToolCallProgress<P>,
): Promise<ToolResult<Output>>
```

### 5.4.2 ToolUseContext 上下文

```typescript
export type ToolUseContext = {
  options: {
    commands: Command[]
    tools: Tools
    mainLoopModel: string
    mcpClients: MCPServerConnection[]
    // ... 更多配置
  }
  abortController: AbortController
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  // ... 更多上下文
}
```

## 5.5 工具渲染系统

### 5.5.1 多阶段渲染

```typescript
// 工具调用时的 UI
renderToolUseMessage(input, options): React.ReactNode

// 工具执行中的进度 UI
renderToolUseProgressMessage(progressMessages, options): React.ReactNode

// 工具结果渲染
renderToolResultMessage(content, progress, options): React.ReactNode

// 工具被拒绝时的 UI
renderToolUseRejectedMessage(input, options): React.ReactNode

// 工具执行错误时的 UI
renderToolUseErrorMessage(result, options): React.ReactNode
```

### 5.5.2 分组渲染

```typescript
// 非 verbose 模式下，多个并行工具调用可以分组显示
renderGroupedToolUse?(toolUses, options): React.ReactNode | null
```

## 5.6 延迟加载与 ToolSearch

### 5.6.1 工具延迟机制

```typescript
// 工具可被标记为延迟加载
readonly shouldDefer?: boolean

// 只有被 tool_reference 发现过的工具才会发送给 API
const discoveredToolNames = extractDiscoveredToolNames(messages)
filteredTools = tools.filter(tool => {
  if (!deferredToolNames.has(tool.name)) return true
  if (toolMatchesName(tool, TOOL_SEARCH_TOOL_NAME)) return true
  return discoveredToolNames.has(tool.name)
})
```

**设计亮点**：
- 解决工具数量限制（API 对工具数量有限制）
- 只有真正被使用的工具才发送到 API

## 5.7 设计模式总结

| 模式 | 应用 |
|------|------|
| **Factory** | `buildTool()` 填充默认值 |
| **Strategy** | 工具权限组件映射 `permissionComponentForTool()` |
| **Dead Code Elimination** | `feature()` 条件编译 |
| **延迟加载** | 运行时 `require()` 打破循环依赖 |
| **缓存优化** | 内置工具保持连续前缀 |
| **fail-closed** | 默认值采用保守假设 |

## 5.8 关键源码位置

- `src/Tool.ts:1-793` — Tool 类型定义和 buildTool 工厂
- `src/tools.ts:1-390` — 工具注册和过滤逻辑
- `src/utils/permissions/permissions.ts` — 权限规则处理
- `src/services/api/claude.ts` — API 请求中的工具过滤