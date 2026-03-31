# 第 9 章：关键架构决策与优化

## 9.1 运行时选择：Bun 而非 Node.js

### 决策内容

- **运行时**：Bun（不是 Node.js）
- **构建**：`bun build` 输出单文件 bundle
- **模块系统**：ESM + TSX + react-jsx

### 设计理由

```typescript
// cli.tsx 入口点注入 polyfill
import { feature } from 'bun:bundle'

// feature() 在此版本中始终返回 false
globalThis.MACRO = {
  VERSION: '0.0.0',
  BUILD_TIME: Date.now().toString(),
  BUILD_TARGET: 'bun',
  BUILD_ENV: 'development',
  INTERFACE_TYPE: 'terminal',
}
```

### 影响

- 直接使用 Bun API，无需 Node.js 兼容层
- 构建产物 ~25MB 单文件
- tsconfig 使用 `bun:bundle` 模块

---

## 9.2 特性标志系统：feature() 恒为 false

### 决策内容

所有 `feature('FLAG_NAME')` 调用在 bun build 时求值，但在 dev 模式下 polyfill 为始终返回 `false`。

### 实现方式

```typescript
// cli.tsx 入口点
function feature(flag: string): boolean {
  // 所有标志默认关闭
  return false
}

// 编译时由 bun:bundle 替换
const COORDINATOR_MODE = feature('COORDINATOR_MODE') // → false
```

### 影响

| 影响 | 说明 |
|------|------|
| Dead code | 所有特性标志代码都是死代码 |
| 简化构建 | 无需复杂的条件编译配置 |
| 可预测 | 所有功能基于统一规则开关 |

---

## 9.3 简约状态管理：手写 Store 而非依赖库

### 决策内容

手动实现 Zustand 风格的状态管理，不依赖外部状态库。

### 实现方式

```typescript
// 35 行实现
export function createStore<T>(initialState: T, onChange?: OnChange<T>): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,
    setState: (updater) => {
      const next = updater(state)
      if (Object.is(next, state)) return  // 浅比较
      state = next
      for (const listener of listeners) listener()
    },
    subscribe: (listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

### 设计优势

- **零依赖**：不引入额外依赖
- **轻量**：35 行代码 vs Zustand 数千行
- **足够**：满足单例 Store 需求

---

## 9.4 React 订阅：使用 useSyncExternalStore

### 决策内容

使用 React 18 官方推荐的 `useSyncExternalStore` 进行状态订阅。

### 实现方式

```typescript
export function useAppState<R>(selector: (state: AppState) => R): R {
  const store = useAppStore()
  const get = () => selector(store.getState())
  return useSyncExternalStore(store.subscribe, get, get)
}
```

### 设计优势

- **React 18 兼容**：官方推荐的订阅方式
- **SSR 兼容**：支持 getServerSnapshot
- **性能**：选择器细粒度订阅，避免整体 re-render

---

## 9.5 早写入设计：Transcript 提前持久化

### 决策内容

在 API 响应前就写入 transcript，支持 kill-mid-request 恢复。

### 实现方式

```typescript
// QueryEngine.ts:453-466
// 用户消息入队时立即写入，不等待 API 响应
if (persistSession && messagesFromUserInput.length > 0) {
  const transcriptPromise = recordTranscript(messages)
  if (isBareMode()) {
    void transcriptPromise  // fire-and-forget
  } else {
    await transcriptPromise
    await flushSessionStorage()
  }
}
```

### 设计优势

- **恢复能力**：kill-mid-request 后仍可 `--resume`
- **用户体验**：写入 ~4ms（SSD），可接受延迟
- **容错**：bare 模式下 fire-and-forget

---

## 9.6 UUID 锚点：消息分页策略

### 决策内容

消息分页使用 UUID 而非索引作为锚点，防止压缩时内容跳动。

### 实现方式

```typescript
// Messages.tsx
export function computeSliceStart(collapsed, anchorRef, cap = 200, step = 50) {
  const anchor = anchorRef.current
  const anchorIdx = anchor
    ? collapsed.findIndex(m => m.uuid === anchor.uuid)
    : -1

  let start = anchorIdx >= 0
    ? anchorIdx
    : anchor
      ? Math.min(anchor.idx, Math.max(0, collapsed.length - cap))
      : 0

  // 超过阈值才推进
  if (collapsed.length - start > cap + step) {
    start = collapsed.length - cap
  }

  // 刷新锚点
  const msgAtStart = collapsed[start]
  if (msgAtStart) {
    anchorRef.current = { uuid: msgAtStart.uuid, idx: start }
  }

  return start
}
```

### 设计优势

- **稳定性**：分组/压缩后内容不跳动
- **回退**：锚点丢失时回退到索引
- **边界**：200 条上限 + 50 步进

---

## 9.7 Provider 抽象：统一入口支持多后端

### 决策内容

单一入口 `getAPIProvider()` 支持 Anthropic/Bedrock/Vertex/Azure。

### 实现方式

```typescript
// providers.ts
export type APIProvider = 'firstParty' | 'bedrock' | 'vertex' | 'foundry'

export function getAPIProvider(): APIProvider {
  return isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)
    ? 'bedrock'
    : isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX)
      ? 'vertex'
      : isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY)
        ? 'foundry'
        : 'firstParty'
}

// 差异化处理
if (useToolSearch && getAPIProvider() !== 'bedrock') {
  betas.push(getToolSearchBetaHeader())
}
```

### 设计优势

- **环境变量驱动**：无运行时检测开销
- **差异化**：各模块按需使用
- **可扩展**：新增 Provider 只需扩展入口

---

## 9.8 延迟工具加载：ToolSearch 机制

### 决策内容

只有被 `tool_reference` 发现过的工具才发送给 API，解决工具数量限制。

### 实现方式

```typescript
// claude.ts
const deferredToolNames = new Set<string>()
// 延迟工具名称收集
for (const t of tools) {
  if (isDeferredTool(t)) deferredToolNames.add(t.name)
}

// 实际过滤
filteredTools = tools.filter(tool => {
  if (!deferredToolNames.has(tool.name)) return true  // 非延迟工具直接通过
  if (toolMatchesName(tool, TOOL_SEARCH_TOOL_NAME)) return true  // ToolSearch 本身总是包含
  return discoveredToolNames.has(tool.name)  // 只有被发现的才发送
})
```

### 设计优势

- **突破限制**：解决 API 工具数量限制
- **按需加载**：只发送实际使用的工具
- **优化成本**：减少 token 消耗

---

## 9.9 缓存优化：内置工具连续前缀

### 决策内容

内置工具保持为连续前缀，用于 prompt cache 优化。

### 实现方式

```typescript
// tools.ts
export function assembleToolPool(permissionContext, mcpTools): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)

  // 按名称排序，内置工具保持为连续前缀
  const byName = (a, b) => a.name.localeCompare(b.name)
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name'
  )
}
```

### 设计理由

- 缓存策略在服务端放置全局断点
- 内置工具连续排列避免缓存失效
- MCP 工具追加在后面

---

## 9.10 关键架构决策汇总

| 决策 | 实现方式 | 影响 |
|------|----------|------|
| Bun 运行时 | 不使用 Node.js 兼容层 | 25MB 单文件 bundle |
| feature() 恒为 false | polyfill 始终返回 false | 所有特性代码是 dead code |
| 手写 Store | 35 行 Zustand 风格 | 零外部依赖 |
| useSyncExternalStore | React 18 官方推荐 | SSR 兼容 |
| 早写入 Transcript | API 响应前持久化 | 支持 kill-mid-request 恢复 |
| UUID 锚点 | 使用 uuid 而非索引 | 防止压缩后内容跳动 |
| Provider 抽象 | 单一入口 getAPIProvider() | 支持多后端 |
| 延迟工具加载 | ToolSearch 机制 | 突破 API 限制 |
| 缓存优化 | 连续前缀策略 | 优化 prompt cache |