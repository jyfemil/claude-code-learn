# 第 7 章：状态管理与响应式设计

## 7.1 简约 Store 实现

### 7.1.1 核心实现

```typescript
type Listener = () => void
type OnChange<T> = (args: { newState: T; oldState: T }) => void

export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}

export function createStore<T>(
  initialState: T,
  onChange?: OnChange<T>,
): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,

    setState: (updater: (prev: T) => T) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return  // 浅比较优化
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },

    subscribe: (listener: Listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

**设计亮点**：
- 手写 Zustand 风格实现，不依赖外部库
- 浅比较优化：`Object.is(next, prev)` 避免不必要的更新
- 订阅返回取消函数，便于清理

### 7.1.2 与 React 集成

```typescript
export function useAppState<R>(selector: (state: AppState) => R): R {
  const store = useAppStore()
  const get = () => selector(store.getState())
  return useSyncExternalStore(store.subscribe, get, get)
}

// 使用示例
const verbose = useAppState(s => s.verbose)
const mcp = useAppState(s => s.mcp.tools)
const messages = useAppState(s => s.messages)
```

**设计亮点**：
- `useSyncExternalStore` 是 React 18 官方推荐的订阅方式
- 选择性订阅：只订阅关心的状态切片，避免整体 re-render

## 7.2 AppState 结构

### 7.2.1 核心状态

```typescript
export type AppState = {
  // 对话
  messages: Message[]
  compactMessages: Message[]

  // 工具与权限
  tools: Tools
  toolPermissionContext: ToolPermissionContext
  toolUseConfirmQueue: ToolUseConfirm<any>[]

  // MCP
  mcp: {
    servers: MCPServerConnection[]
    tools: Tools
    resources: Map<string, ServerResource[]>
  }

  // UI 状态
  theme: ThemeName
  verbose: boolean
  isFirstTurn: boolean
  paused: boolean
  loading: boolean

  // 任务系统
  tasks: Task[]

  // 文件历史
  fileHistory: FileHistoryState

  // ... 更多
}
```

### 7.2.2 状态分区

```typescript
// 消息相关
messages: Message[]
compactMessages: Message[]

// MCP 连接
mcp: {
  servers: MCPServerConnection[]
  tools: Tools
  resources: Map<string, ServerResource[]>
}

// 权限上下文（频繁更新）
toolPermissionContext: ToolPermissionContext
toolUseConfirmQueue: ToolUseConfirm<any>[]
```

## 7.3 响应式设计模式

### 7.3.1 原子化订阅

```typescript
// 每次调用 useAppState 都会创建新的订阅
const verbose = useAppState(s => s.verbose)           // 只关心 verbose
const loading = useAppState(s => s.loading)           // 只关心 loading
const tools = useAppState(s => s.tools)                // 只关心工具

// 改变 verbose 不会触发 loading 或 tools 的 re-render
```

**设计亮点**：
- 细粒度订阅，避免整体状态变化导致的级联 re-render

### 7.3.2 useSyncExternalStore 模式

```typescript
function useSyncExternalStore<T>(
  subscribe: (callback: () => void) => () => void,
  getSnapshot: () => T,
  getServerSnapshot?: () => T,
): T
```

**React 18 兼容性**：
- 替代 deprecated 的 `unstable_batchedUpdates`
- 支持 concurrent rendering
- SSR 兼容（通过 getServerSnapshot）

## 7.4 状态更新模式

### 7.4.1 函数式更新

```typescript
setAppState(prev => ({
  ...prev,
  messages: [...prev.messages, newMessage],
}))

// 嵌套更新
setAppState(prev => ({
  ...prev,
  toolPermissionContext: {
    ...prev.toolPermissionContext,
    alwaysAllowRules: {
      ...prev.toolPermissionContext.alwaysAllowRules,
      command: allowedTools,
    },
  },
}))
```

### 7.4.2 批量更新

```typescript
// 每次 setState 都会同步所有监听器
// 如果需要批量更新，可以使用 requestIdleCallback 或 queueMicrotask
```

## 7.5 状态持久化

### 7.5.1 Transcript 持久化

```typescript
// QueryEngine.ts
if (persistSession && messagesFromUserInput.length > 0) {
  const transcriptPromise = recordTranscript(messages)
  await transcriptPromise
  await flushSessionStorage()
}
```

### 7.5.2 文件历史快照

```typescript
if (fileHistoryEnabled() && persistSession) {
  messagesFromUserInput
    .filter(messageSelector().selectableUserMessagesFilter)
    .forEach(message => {
      void fileHistoryMakeSnapshot(
        (updater) => setAppState(prev => ({ ...prev, fileHistory: updater(prev.fileHistory) })),
        message.uuid,
      )
    })
}
```

## 7.6 状态隔离与子代理

### 7.6.1 createSubagentContext

```typescript
// 子代理创建独立的上下文，状态管理不同
function createSubagentContext(parentContext, agentType) {
  return {
    // setAppState 变为 no-op，状态不写回主 store
    setAppState: () => {},
    // 使用独立的 message 数组
    messages: [],
    // ...
  }
}
```

### 7.6.2 localDenialTracking

```typescript
// 子代理有独立的拒绝追踪
toolUseContext: {
  // 权限拒绝累积，用于阈值判断
  localDenialTracking?: DenialTrackingState
}
```

## 7.7 性能优化

### 7.7.1 浅比较优化

```typescript
setState: (updater) => {
  const next = updater(prev)
  if (Object.is(next, prev)) return  // 相同引用直接返回
  state = next
  // ...
}
```

### 7.7.2 保守的 Memo 比较

```typescript
// MessageRow.tsx
export function areMessageRowPropsEqual(prev, next): boolean {
  if (prev.message !== next.message) return false
  if (prev.screen !== next.screen) return false

  // 流式/完成状态判断
  const isStreaming = isMessageStreaming(prev.message, prev.streamingToolUseIDs)
  const isResolved = allToolsResolved(prev.message, prev.lookups.resolvedToolUseIDs)

  if (isStreaming || !isResolved) return false
  return true  // 静态消息可跳过重渲染
}
```

## 7.8 设计模式总结

| 模式 | 应用 |
|------|------|
| **Observer** | listeners 订阅机制 |
| **Atomic Subscription** | 细粒度 selector 订阅 |
| **Fail-fast** | 浅比较优化 `Object.is()` |
| **Dependency Injection** | config 注入状态访问 |
| **Copy-on-write** | 函数式更新返回新对象 |

## 7.9 关键源码位置

- `src/state/store.ts:1-35` — 简约 Store 实现
- `src/state/AppState.tsx` — AppState 类型和 Provider
- `src/components/MessageRow.tsx` — 消息渲染的 memo 优化
- `src/utils/sessionStorage.ts` — Transcript 持久化