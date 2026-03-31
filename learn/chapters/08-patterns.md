# 第 8 章：核心设计模式总结

## 8.1 AsyncGenerator 流水线模式

### 8.1.1 核心思想

整个查询循环是一个 `AsyncGenerator`，实现**边接收边处理**的流式交互，而非等待完整响应后再处理。

### 8.1.2 应用场景

**QueryEngine 核心循环**：

```typescript
// QueryEngine.ts:679-690
for await (const message of query({
  messages,
  systemPrompt,
  userContext,
  canUseTool: wrappedCanUseTool,
  toolUseContext: processUserInputContext,
  fallbackModel,
  querySource: 'sdk',
  maxTurns,
  taskBudget,
})) {
  // 边接收边 yield，无等待
  switch (message.type) {
    case 'assistant':
      this.mutableMessages.push(msg)
      yield* normalizeMessage(msg)
      break
    // ...
  }
}
```

### 8.1.3 设计优势

| 优势 | 说明 |
|------|------|
| **低延迟** | 无需等待完整响应，首字节即可开始处理 |
| **实时反馈** | 工具调用可与模型输出并行执行 |
| **内存效率** | 不需要缓存完整响应 |

---

## 8.2 依赖注入模式

### 8.2.1 核心思想

通过 config 对象注入所有依赖，而非在类内部直接创建或引用全局状态。

### 8.2.2 应用场景

**QueryEngine 配置**：

```typescript
export type QueryEngineConfig = {
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  readFileCache: FileStateCache
  // ... 更多依赖
}
```

### 8.2.3 设计优势

- **可测试性**：测试时可以注入 mock 实现
- **可替换性**：不同环境（REPL/SDK）可以注入不同实现
- **解耦**：类不直接依赖全局状态

---

## 8.3 工厂模式（buildTool）

### 8.3.1 核心思想

使用工厂函数创建工具对象，自动填充默认值。

### 8.3.2 应用场景

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

### 8.3.3 设计优势

- **fail-closed 原则**：默认值采用保守假设（不安全、不只读）
- **一致性**：所有工具通过同一工厂创建，接口统一
- **简化调用**：调用者无需处理 `?.() ?? default`

---

## 8.4 观察者模式（Store 订阅）

### 8.4.1 核心思想

使用 Set 存储监听器，状态变化时通知所有订阅者。

### 8.4.2 应用场景

```typescript
// store.ts
export function createStore<T>(initialState: T, onChange?: OnChange<T>): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,

    setState: (updater) => {
      const next = updater(state)
      if (Object.is(next, state)) return  // 浅比较优化
      state = next
      onChange?.({ newState: next, oldState: state })
      for (const listener of listeners) listener()
    },

    subscribe: (listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

### 8.4.3 设计优势

- **去重**：Set 自动去重
- **清理**：返回取消函数便于资源清理
- **批量通知**：一次性通知所有监听器

---

## 8.5 Pipeline 组合模式

### 8.5.1 核心思想

通过函数组合形成数据处理流水线。

### 8.5.2 应用场景

**消息处理流水线**：

```typescript
const collapsed = collapseBackgroundBashNotifications(
  collapseHookSummaries(
    collapseTeammateShutdowns(
      collapseReadSearchGroups(
        applyGrouping(
          reorderMessagesInUI(messagesToShow),
          tools,
          verbose
        ),
        tools
      )
    ),
    verbose
  )
)
```

### 8.5.3 设计优势

- **可组合**：每个函数独立，可自由组合
- **可测试**：每个函数可单独测试
- **可扩展**：新增处理步骤只需添加新函数

---

## 8.6 Dead Code Elimination（条件编译）

### 8.6.1 核心思想

基于编译时常量（`feature()`）和条件导入实现 tree-shaking。

### 8.6.2 应用场景

```typescript
// 基于 feature() 的条件导入
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null

// 基于环境变量的条件加载
const REPLTool = process.env.USER_TYPE === 'ant'
  ? require('./tools/REPLTool/REPLTool.js').REPLTool
  : null
```

### 8.6.3 设计优势

- **构建优化**：未使用的代码不进入最终 bundle
- **特性切换**：同一代码库支持不同功能集
- **运行时零开销**：条件在编译时求值

---

## 8.7 延迟加载模式

### 8.7.1 核心思想

运行时动态 require，打破循环依赖。

### 8.7.2 应用场景

```typescript
// 打破循环依赖
const getPowerShellTool = () => {
  if (!isPowerShellToolEnabled()) return null
  return require('./tools/PowerShellTool/PowerShellTool.js').PowerShellTool
}

// 工具延迟加载（突破 API 限制）
const deferredToolNames = new Set<string>()
// 只有被 tool_reference 发现过的工具才发送给 API
filteredTools = tools.filter(tool => {
  if (!deferredToolNames.has(tool.name)) return true
  return discoveredToolNames.has(tool.name)
})
```

### 8.7.3 设计优势

- **解决循环依赖**：运行时加载打破编译时循环
- **突破限制**：延迟加载解决 API 工具数量限制

---

## 8.8 状态机模式（QueryGuard）

### 8.8.1 核心思想

用单一状态机替代多个布尔标志。

### 8.8.2 应用场景

```typescript
// REPL.tsx
const queryGuard = React.useRef(new QueryGuard()).current
const isQueryActive = React.useSyncExternalStore(
  queryGuard.subscribe,
  queryGuard.getSnapshot
)
```

### 8.8.3 设计优势

- **一致性**：避免多个状态标志不同步
- **可观测**：状态变化可追踪
- **可测试**：状态机逻辑可单独测试

---

## 8.9 设计模式汇总

| 模式 | 核心类/函数 | 章节 |
|------|------------|------|
| AsyncGenerator 流水线 | query() | 3 |
| 依赖注入 | QueryEngineConfig | 3 |
| 工厂模式 | buildTool() | 5 |
| 观察者模式 | createStore() | 7 |
| Pipeline 组合 | collapse* 函数 | 6 |
| 条件编译 | feature() | 5 |
| 延迟加载 | require() | 5 |
| 状态机 | QueryGuard | 6 |