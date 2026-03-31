# 第 10 章：API 客户端

> 本章聚焦 `services/api/claude.ts`，这是与 Claude API 交互的**底层客户端**

## 10.1 claude.ts 核心定位

### 10.1.1 职责

`claude.ts` 是**底层 API 客户端**，被 `query.ts` 调用。负责：
- 多 Provider 支持（Anthropic/Bedrock/Vertex/Azure）
- 请求构建（系统提示、工具定义、消息序列化）
- 流式事件处理
- 工具搜索集成

### 10.1.2 核心函数

```typescript
// claude.ts
export async function* queryModelWithStreaming({
  messages,
  systemPrompt,
  tools,
  model,
  // ... 更多参数
}): AsyncGenerator<StreamEvent | AssistantMessage | SystemAPIErrorMessage>

export async function queryModelWithoutStreaming(...)
export async function queryWithModel(...)
```

---

## 10.2 多 Provider 支持

### 10.2.1 Provider 抽象

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
```

### 10.2.2 差异化处理

不同 Provider 使用不同的参数格式：

```typescript
// Beta headers 差异化处理
if (getAPIProvider() !== 'bedrock') {
  // Bedrock 使用 extraBodyParams
  betas.push(getToolSearchBetaHeader())
}
```

---

## 10.3 请求构建

### 10.3.1 工具过滤与延迟加载

```typescript
// claude.ts:1130-1165
const deferredToolNames = new Set<string>()
for (const t of tools) {
  if (isDeferredTool(t)) deferredToolNames.add(t.name)
}

// 延迟工具过滤：只有被发现的工具才发送给 API
filteredTools = tools.filter(tool => {
  if (!deferredToolNames.has(tool.name)) return true  // 非延迟工具直接通过
  if (toolMatchesName(tool, TOOL_SEARCH_TOOL_NAME)) return true  // ToolSearch 本身总是包含
  return discoveredToolNames.has(tool.name)  // 只有被发现的才发送
})
```

### 10.3.2 延迟加载解决的问题

- **API 限制**：Claude API 对工具数量有限制
- **按需加载**：只发送实际使用的工具
- **成本优化**：减少 token 消耗

---

## 10.4 流式事件处理

### 10.4.1 queryModelWithStreaming

```typescript
// claude.ts:753-819
export async function* queryModelWithStreaming({
  messages,
  systemPrompt,
  tools,
  model,
  // ...
}): AsyncGenerator<StreamEvent | AssistantMessage | SystemAPIErrorMessage> {
  const stream = await makeRequestWithRetries(...)

  for await (const event of stream) {
    switch (event.type) {
      case 'message_start':
        // 消息开始，重置 usage
        currentMessageUsage = updateUsage(currentMessageUsage, event.message.usage)
        break

      case 'content_block_start':
        // 开始新块
        break

      case 'content_block_delta':
        // 处理增量（文本或工具输入）
        // 实时更新 assistant message
        break

      case 'content_block_stop':
        // 块完成
        break

      case 'message_delta':
        // 更新 usage 和 stop_reason
        currentMessageUsage = updateUsage(currentMessageUsage, event.usage)
        break

      case 'message_stop':
        // 消息结束，累积 usage
        totalUsage = accumulateUsage(totalUsage, currentMessageUsage)
        break
    }

    // yield 事件供上层处理
    yield event as StreamEvent
  }
}
```

### 10.4.2 完全流式处理

- 边接收边处理，无需等待完整响应
- 实时更新 assistant message
- 增量更新 usage 统计

---

## 10.5 消息序列化

### 10.5.1 用户消息转换

```typescript
// claude.ts:589-634
export function userMessageToMessageParam(message: UserMessage): MessageParam {
  return {
    role: 'user',
    content: message.message.content,  // ContentBlockParam[]
  }
}
```

### 10.5.2 助手消息转换

```typescript
export function assistantMessageToMessageParam(message: AssistantMessage): MessageParam {
  return {
    role: 'assistant',
    content: message.message.content,  // ContentBlock[]
  }
}
```

---

## 10.6 工具搜索集成

### 10.6.1 ToolSearch 机制

```typescript
// 当启用 ToolSearch 时，部分工具会被标记为 deferred
// 只有被 tool_reference 发现后才发送给 API

// 判断是否启用 ToolSearch
const useToolSearch = isToolSearchEnabled(options.model, {
  isAgenticQuery: options.isAgenticQuery,
  toolCount: tools.length,
})
```

### 10.6.2 动态工具加载

```typescript
// 提取已发现的工具名称
const discoveredToolNames = extractDiscoveredToolNames(messages)

// 只有被发现的延迟工具才会被包含在请求中
const includedDeferredTools = filteredTools.filter(t =>
  deferredToolNames.has(t.name)
).length
```

---

## 10.7 设计模式

| 模式 | 应用 |
|------|------|
| **Provider 抽象** | 单一入口 getAPIProvider() 支持多后端 |
| **延迟加载** | 延迟工具按需发送给 API |
| **流式处理** | AsyncGenerator 边接收边 yield |
| **差异化处理** | 各 Provider 使用不同参数格式 |
| **工具搜索** | tool_reference 发现机制 |

---

## 10.8 关键源码位置

- `src/utils/model/providers.ts:6` — getAPIProvider 入口
- `src/services/api/claude.ts:753` — queryModelWithStreaming
- `src/services/api/claude.ts:1130-1165` — 延迟工具过滤
- `src/services/api/claude.ts:589-634` — 消息序列化