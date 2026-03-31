# 第 4 章：核心查询循环（Agent Loop）

> 本章聚焦 `query.ts` 核心查询循环，这是 Claude Code 的**代理循环引擎**

## 4.1 query.ts 核心定位

### 4.1.1 职责

`query.ts` 是**底层查询函数**，被 `QueryEngine.submitMessage()` 调用。负责：
- 与 Claude API 的直接交互（流式调用）
- 工具调用的执行循环
- 错误恢复与重试

它是整个 CLI 的**核心循环引擎**，驱动 Agent 与模型的交互。

### 4.1.2 函数签名

```typescript
// query.ts:219-239
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  | StreamEvent        // 流式事件（message_start, content_block_delta 等）
  | RequestStartEvent  // 请求开始事件
  | Message            // 消息（assistant, user, system, progress 等）
  | TombstoneMessage  // 墓碑消息（用于删除孤儿消息）
  | ToolUseSummaryMessage,  // 工具调用摘要
  Terminal            // 最终结果
> {
  const consumedCommandUuids: string[] = []
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  // ... 通知命令生命周期
  return terminal
}
```

---

## 4.2 queryLoop 主循环结构

### 4.2.1 状态管理

```typescript
// query.ts:268-279
let state: State = {
  messages: params.messages,           // 对话历史
  toolUseContext: params.toolUseContext, // 工具执行上下文
  maxOutputTokensOverride: undefined,  // 输出 token 限制覆盖
  autoCompactTracking: undefined,      // 自动压缩追踪
  stopHookActive: undefined,            // stop hook 状态
  maxOutputTokensRecoveryCount: 0,     // max_tokens 恢复计数
  hasAttemptedReactiveCompact: false,  // 是否尝试过响应式压缩
  turnCount: 1,                         // 当前 turn 数
  pendingToolUseSummary: undefined,     // 待处理的工具摘要
  transition: undefined,                // 状态转换
}
```

### 4.2.2 主循环

```typescript
// query.ts:307-329
while (true) {
  // 每轮迭代解构状态
  let { toolUseContext } = state
  const {
    messages,
    autoCompactTracking,
    maxOutputTokensRecoveryCount,
    // ...
  } = state

  // 1. 压缩检查
  const { compactionResult, consecutiveFailures } = await deps.autocompact(...)
  if (compactionResult) {
    // 压缩成功后继续（使用压缩后的消息）
    state = { ...state, messages: postCompactMessages, ... }
    continue
  }

  // 2. API 调用循环
  try {
    while (attemptWithFallback) {
      attemptWithFallback = false
      try {
        for await (const message of deps.callModel({ ... })) {
          // 流式处理每条消息
          if (message.type === 'assistant') {
            // 提取工具调用块
            const msgToolUseBlocks = filterToolUseBlocks(message)
            if (msgToolUseBlocks.length > 0) {
              toolUseBlocks.push(...msgToolUseBlocks)
              needsFollowUp = true
            }
          }
        }
      } catch (innerError) {
        // 错误处理与恢复
      }
    }
  } finally {
    // 清理资源
  }

  // 3. 工具执行循环
  if (needsFollowUp) {
    const toolUpdates = streamingToolExecutor
      ? streamingToolExecutor.getRemainingResults()
      : runTools(toolUseBlocks, ...)

    for await (const update of toolUpdates) {
      if (update.message) {
        yield update.message
        toolResults.push(...normalizeMessagesForAPI(...))
      }
      if (update.newContext) {
        toolUseContext = { ...update.newContext, queryTracking }
      }
    }
  }
}
```

---

## 4.3 流式响应处理

### 4.3.1 实时工具调用提取

```typescript
// query.ts - 流式处理循环内
for await (const message of deps.callModel({ ... })) {
  if (message.type === 'assistant') {
    assistantMessages.push(assistantMessage)

    // 实时提取 tool_use 块，无需等待完整响应
    const msgToolUseBlocks = message.message.content.filter(
      (block): block is ToolUseBlock => block.type === 'tool_use'
    )
    if (msgToolUseBlocks.length > 0) {
      toolUseBlocks.push(...msgToolUseBlocks)
      needsFollowUp = true
    }
  }
}
```

### 4.3.2 流式事件处理

```typescript
switch (event.type) {
  case 'content_block_start':
    // 开始新块
    break
  case 'content_block_delta':
    // 处理增量文本/工具输入
    break
  case 'content_block_stop':
    // 块完成
    break
  case 'message_delta':
    // 更新 usage 和 stop_reason
    break
  case 'message_stop':
    // 消息结束
    break
}
```

---

## 4.4 工具执行循环

### 4.4.1 流式工具执行器

```typescript
// 工具执行两种模式：
// 1. StreamingToolExecutor - 允许工具与模型输出并行执行
// 2. runTools - 等待模型完全响应后执行

const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()
  : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)

for await (const update of toolUpdates) {
  if (update.message) {
    // yield 工具结果消息
    yield update.message

    // 标准化后追加到消息历史
    toolResults.push(...normalizeMessagesForAPI(...))
  }

  if (update.newContext) {
    // 更新工具执行上下文
    updatedToolUseContext = { ...update.newContext, queryTracking }
  }
}
```

---

## 4.5 错误处理与恢复

### 4.5.1 Fallback 模型切换

```typescript
// query.ts - Fallback 触发
if (innerError instanceof FallbackTriggeredError && fallbackModel) {
  currentModel = fallbackModel
  attemptWithFallback = true

  // 清理并重试
  yield* yieldMissingToolResultBlocks(assistantMessages, 'Model fallback triggered')
  // ...
  continue
}
```

### 4.5.2 流式回退（Tombstone 机制）

```typescript
// query.ts:712-741
if (streamingFallbackOccured) {
  // Yield tombstones for orphaned messages
  // 防止 partial thinking blocks 导致 API 错误
  for (const msg of assistantMessages) {
    yield { type: 'tombstone' as const, message: msg }
  }

  // 清空状态
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

### 4.5.3 max_output_tokens 恢复

```typescript
// 多级恢复策略：
// 1. 首次触发 -> 升级到 64k
// 2. 再次触发 -> 注入恢复消息
// 3. 超过阈值 -> 报告错误
if (isMaxOutputTokensError(error)) {
  if (maxOutputTokensRecoveryCount === 0) {
    maxOutputTokensOverride = 64000
    maxOutputTokensRecoveryCount++
    attemptWithFallback = true
    continue
  }
}
```

---

## 4.6 对话压缩集成

### 4.6.1 autocompact 检查

```typescript
const { compactionResult, consecutiveFailures } = await deps.autocompact(
  messagesForQuery,
  turnCount,
  consecutiveFailures,
  toolUseContext,
)

if (compactionResult) {
  tracking = {
    compacted: true,
    turnId: deps.uuid(),
    turnCounter: 0,
    consecutiveFailures: 0,
  }

  const postCompactMessages = buildPostCompactMessages(compactionResult)
  messagesForQuery = postCompactMessages
  continue  // 继续使用压缩后的消息进行下一轮
}
```

### 4.6.2 压缩后的消息处理

- 压缩成功后 `continue` 继续循环
- 不是重新开始，而是使用压缩后的消息继续
- 保留对话连续性

---

## 4.7 设计模式

| 模式 | 应用 |
|------|------|
| **AsyncGenerator** | 整个 query 是 async generator，实现流式处理 |
| **状态机** | while(true) + state 对象管理多轮对话 |
| **早失败** | 错误 early return，避免深层嵌套 |
| **计数恢复** | maxOutputTokensRecoveryCount 控制重试次数 |
| **墓碑机制** | 流式回退时用 tombstone 清理孤儿消息 |

---

## 4.8 关键源码位置

- `src/query.ts:219-239` — query 函数导出
- `src/query.ts:241-329` — queryLoop 主循环
- `src/query.ts:659-780` — API 调用与流式处理
- `src/query.ts:800-950` — 工具执行循环
- `src/query.ts:700-741` — 流式回退与 tombstone
- `src/query.ts:450-550` — autocompact 集成