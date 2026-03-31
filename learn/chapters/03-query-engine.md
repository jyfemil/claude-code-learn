# 第 3 章：Query Engine 对话引擎

## 3.1 核心职责

`QueryEngine` 是 Claude Code 的核心引擎，负责管理对话生命周期和会话状态。它封装了与 Claude API 的交互逻辑，支持：

- **单轮对话**：`submitMessage()` 处理用户输入并流式返回结果
- **多轮对话**：状态跨 turn 持久化（消息历史、文件缓存、usage 统计）
- **错误恢复**：预算控制、fallback 模型、自动压缩

## 3.2 类结构设计

### 3.2.1 状态封装

```typescript
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]           // 对话历史
  private abortController: AbortController      // 中断控制
  private permissionDenials: SDKPermissionDenial[]  // 权限拒绝记录
  private totalUsage: NonNullableUsage          // token 统计
  private readFileState: FileStateCache        // 文件读取缓存
  private discoveredSkillNames = new Set<string>()    // 本轮发现的技能
  private loadedNestedMemoryPaths = new Set<string>()  // 已加载的记忆路径
}
```

**设计亮点**：
- 所有状态封装在类内部，清晰分离关注点
- `mutableMessages` vs `messages`：内部可变数组 vs 副本（防止外部污染）
- 跨 turn 持久化的状态（文件缓存、技能发现）无需每次重建

### 3.2.2 配置注入

```typescript
export type QueryEngineConfig = {
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  // ... 更多配置
}
```

**依赖注入模式**：将依赖（工具、状态访问、缓存）通过 config 注入，便于测试和替换实现。

## 3.3 核心流程：submitMessage

### 3.3.1 初始化阶段

```typescript
async *submitMessage(prompt, options?) {
  // 1. 环境设置
  this.discoveredSkillNames.clear()  // 清理每轮状态
  setCwd(cwd)

  // 2. 构建系统提示
  const { defaultSystemPrompt, userContext, systemContext }
    = await fetchSystemPromptParts({ tools, mainLoopModel, ... })

  // 3. 处理用户输入（slash 命令等）
  const { messages: messagesFromUserInput, shouldQuery, allowedTools }
    = await processUserInput({ input: prompt, ... })
}
```

### 3.3.2 消息持久化（关键设计）

```typescript
// 在 API 响应前就写入 transcript，支持 kill-mid-request 恢复
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

**设计亮点**：
- Transcript 在用户消息入队时就写入，而非等待 API 响应
- 如果在 API 响应前 kill 进程，session 仍可恢复（`--resume` 不会因"无对话"失败）

### 3.3.3 查询循环

```typescript
for await (const message of query({ messages, systemPrompt, ... })) {
  switch (message.type) {
    case 'assistant':
      this.mutableMessages.push(msg)
      yield* normalizeMessage(msg)
      break
    case 'tool_use_summary':
      yield { type: 'tool_use_summary', summary: msg.summary, ... }
      break
    // ... 更多消息类型处理
  }
}
```

**流式处理**：使用 `AsyncGenerator` 边接收边 yield，实现真正的流式交互。

## 3.4 对话压缩机制

### 3.4.1 多层压缩策略

Claude Code 支持多层压缩（由轻到重）：

1. **snip**：删除特定消息
2. **microcompact**：精简消息内容
3. **context_collapse**：合并相邻消息
4. **autocompact**：自动触发完整压缩

### 3.4.2 压缩边界处理

```typescript
case 'system':
  if (msg.subtype === 'compact_boundary' && msg.compactMetadata) {
    // 释放压缩前的消息用于 GC
    const mutableBoundaryIdx = this.mutableMessages.length - 1
    if (mutableBoundaryIdx > 0) {
      this.mutableMessages.splice(0, mutableBoundaryIdx)
    }

    yield {
      type: 'system',
      subtype: 'compact_boundary',
      compact_metadata: toSDKCompactMetadata(compactMsg.compactMetadata),
    }
  }
  break
```

**内存管理**：压缩后立即 splice 释放旧消息，避免长对话内存泄漏。

## 3.5 错误与预算控制

### 3.5.1 USD 预算检查

```typescript
if (maxBudgetUsd !== undefined && getTotalCost() >= maxBudgetUsd) {
  yield {
    type: 'result',
    subtype: 'error_max_budget_usd',
    errors: [`Reached maximum budget ($${maxBudgetUsd})`],
  }
  return
}
```

### 3.5.2 Turn 限制

```typescript
if (attachment.type === 'max_turns_reached') {
  yield {
    type: 'result',
    subtype: 'error_max_turns',
    errors: [`Reached maximum number of turns (${maxTurns})`],
  }
  return
}
```

## 3.6 设计模式总结

| 模式 | 应用 |
|------|------|
| **AsyncGenerator** | 边接收边处理的流式交互 |
| **依赖注入** | config 注入所有依赖，便于测试 |
| **状态封装** | 类内部维护所有会话状态 |
| **早写入** | Transcript 在 API 响应前持久化，支持恢复 |
| **内存及时释放** | 压缩后立即 splice 旧消息 |

## 3.7 进阶设计亮点

### 3.7.1 流式回退（Tombstone 机制）

当模型请求发生流式回退（从主模型切换到 fallback 模型）时，需要清理"孤儿消息"（partial messages）：

```typescript
// query.ts:712-728
if (streamingFallbackOccured) {
  // Yield tombstones for orphaned messages so they're removed from UI and transcript.
  // These partial messages (especially thinking blocks) have invalid signatures
  // that would cause "thinking blocks cannot be modified" API errors.
  for (const msg of assistantMessages) {
    yield { type: 'tombstone' as const, message: msg }
  }

  assistantMessages.length = 0
  toolResults.length = 0
  toolUseBlocks.length = 0

  // 丢弃失败的流式执行器，创建新的
  if (streamingToolExecutor) {
    streamingToolExecutor.discard()
    streamingToolExecutor = new StreamingToolExecutor(...)
  }
}
```

**设计亮点**：
- Tombstone 是控制消息，用于从 UI 和 transcript 中移除消息
- 防止 partial thinking blocks 导致 "thinking blocks cannot be modified" API 错误

### 3.7.2 错误 Withholding（延迟暴露）

某些可恢复的错误（如 prompt-too-long）不会立即暴露给用户，而是延迟处理：

```typescript
// 错误 withholding 检查 - 需要与 recovery 逻辑一致
// CACHED_MAY_BE_STALE 可能在 5-30s 的流式过程中变化
const mediaRecoveryEnabled = reactiveCompact?.isReactiveCompactEnabled() ?? false
```

**设计亮点**：
- 延迟暴露直到确认无法恢复
- CACHED_MAY_BE_STALE 在流式过程中可能变化，需要与 recovery 逻辑一致

### 3.7.3 消息分页的 UUID 锚点

Messages.tsx 使用 UUID 而非索引作为锚点，防止压缩后内容跳动：

```typescript
// Messages.tsx:315-341
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

**设计亮点**：
- UUID 锚点而非索引，防止分组/压缩时内容跳动
- 锚点丢失时回退到索引，但保留位置信息

## 3.8 关键源码位置

- `src/QueryEngine.ts:186-1202` — QueryEngine 类完整实现
- `src/query.ts:700-740` — 流式回退与 Tombstone 处理
- `src/components/Messages.tsx:315-341` — UUID 锚点分页