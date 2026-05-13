# Week 1 Day 1-2 — Agent Loop：Agent 是怎么跑起来的

> **核心问题**：用户说了一句话，Agent是怎么处理、思考、行动的？整个循环是怎么运作的？

---

## 一、什么是 Agent Loop？

Agent Loop 是 Agent 的"发动机"——一个不断重复的循环：

```
用户输入消息
    ↓
构建对话上下文（加入历史消息、系统提示、记忆等）
    ↓
发给 LLM（大语言模型）
    ↓
LLM 返回：可能是文字回复，也可能是"工具调用请求"
    ↓
如果是工具调用 → 执行工具 → 把结果返回给 LLM → 继续循环
如果是文字回复 → 返回给用户 → 循环结束
```

这就是为什么你跟 Claude Code 对话时，它有时候会"思考一会儿然后执行命令"——那是它在循环里跑工具。

---

## 二、Agent Loop 的核心数据流

```
┌──────────────────────────────────────────────────────────────────┐
│                         query() 主循环                            │
│                        (src/query.ts)                             │
└────────────────────────┬─────────────────────────────────────────┘
                         │
         ┌───────────────┼────────────────┐
         │               │                │
         ▼               ▼                ▼
    ┌─────────┐   ┌──────────────┐  ┌────────────────────────┐
    │消息构建  │   │  上下文压缩   │  │  Token 预算检查         │
    │         │   │(compaction)  │  │(是否超过上下文限制)      │
    └────┬────┘   └──────┬───────┘  └───────────┬────────────┘
         │                │                      │
         └────────────────┼──────────────────────┘
                          ▼
              ┌───────────────────────┐
              │   prependUserContext() │
              │   + fullSystemPrompt   │
              │   = 发送给 LLM 的请求   │
              └───────────┬────────────┘
                          │  API 调用
                          ▼
              ┌───────────────────────┐
              │    LLM 返回流          │
              │  (Streaming)           │
              └───┬───────────────┬────┘
                  │               │
           text / thinking    tool_use 块
                  │               │
                  │               ▼
                  │        ┌────────────┐
                  │        │ 工具执行    │
                  │        │ runTools() │
                  │        └────┬───────┘
                  │             │ 工具结果
                  │             ▼
                  │     加回消息列表
                  │     继续循环 ──────────────┐
                  │                             │
                  ▼                             │
          返回给用户，循环结束 ◀─────────────────┘
```

---

## 三、关键源码位置

| 文件 | 作用 | 重要程度 |
|------|------|---------|
| `src/query.ts` | **主循环**，~1700行，整个Agent Loop的灵魂 | ⭐⭐⭐⭐⭐ |
| `src/query/config.ts` | 查询配置（模型选择、feature gates） | ⭐⭐⭐ |
| `src/query/deps.ts` | 依赖注入（生产依赖） | ⭐⭐ |
| `src/bootstrap/state.ts` | Token预算管理 | ⭐⭐⭐ |
| `src/services/compact/autoCompact.ts` | 自动压缩（当上下文太长时） | ⭐⭐⭐⭐ |
| `src/services/tools/toolOrchestration.ts` | 工具编排执行 | ⭐⭐⭐⭐⭐ |
| `src/Tool.ts` | 工具注册机制 | ⭐⭐⭐⭐ |

---

## 四、逐行理解 query.ts 主循环结构

### 外层结构（query.ts 关键行）

```typescript
// query.ts 第 197 行
export async function* query(params: QueryParams): AsyncGenerator<...> {
  // yield* 是 async generator 的语法
  // 每一次 yield 都是一个"输出事件"（流式文字、工具结果等）
  
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  return terminal  // 循环结束，返回终止原因
}
```

### 核心 while(true) 循环（queryLoop 内）

```typescript
// ~第265行开始
while (true) {
  // ── 1. 消息预处理 ──
  let messagesForQuery = [...getMessagesAfterCompactBoundary(messages)]
  
  // ── 2. 上下文压缩（自动的）────
  const { compactionResult } = await deps.autocompact(
    messagesForQuery, toolUseContext, ...
  )
  if (compactionResult) {
    messagesForQuery = postCompactMessages  // 用压缩后的摘要替换
  }
  
  // ── 3. 构建发给 LLM 的请求 ──
  const fullSystemPrompt = asSystemPrompt(
    appendSystemContext(systemPrompt, systemContext)
  )
  
  // ── 4. 调用 LLM API ──
  for await (const message of deps.callModel({
    messages: prependUserContext(messagesForQuery, userContext),
    systemPrompt: fullSystemPrompt,
    tools: toolUseContext.options.tools,
    ...
  })) {
    // message 是流式返回的片段（可能是 text、thinking、tool_use）
    
    if (message.type === 'tool_use') {
      // 收集所有 tool_use 块
      toolUseBlocks.push(message)
    }
  }
  
  // ── 5. 判断：有没有工具要调？────
  if (toolUseBlocks.length > 0) {
    // 执行工具
    const toolResults = await runTools(toolUseBlocks, toolUseContext)
    // 把结果加回消息列表
    messages = [...messages, ...assistantMessages, ...toolResults]
    continue  // ← 继续下一次循环
  } else {
    // 没有工具了，返回最终的文字回复
    return { reason: 'completed' }
  }
}
```

---

## 五、重要概念解释

### 5.1 消息列表（messages）

Agent 的"记忆"就是一条消息列表：

```
messages = [
  { role: 'system',  content: '你是一个编程助手...' },   // 系统提示
  { role: 'user',    content: '帮我写一个排序算法' },    // 用户说的话
  { role: 'assistant', content: '好的，我来写...' },    // AI 的回复
  { role: 'user',    content: '加个二分查找' },          // 用户又说话
  { role: 'assistant', content: '...' },                 // AI 又回复
  ...
]
```

每次循环，消息列表都会增长——新的用户消息、AI 回复、工具执行结果都加进去。

### 5.2 工具调用（tool_use）

当 LLM 觉得需要执行操作时（比如运行命令、读取文件），它返回一个 `tool_use` 块：

```json
{
  "type": "tool_use",
  "id": "toolu_01XXXX",
  "name": "Bash",
  "input": {
    "command": "ls -la"
  }
}
```

这告诉 Agent："我想要你执行 `ls -la` 这个命令"。

### 5.3 上下文压缩（AutoCompact）

当消息列表太长时，Claude Code 会自动压缩——把一段消息压缩成一个摘要，防止超出 LLM 的上下文限制。这就像清理桌面，把散落的文件归到文件夹里。

### 5.4 Streaming

LLM 的回复是**流式**返回的——一个字一个字地生成，而不是等全部生成完再返回。这样用户能看到 Claude 思考的过程。

---

## 六、伪代码总结

```python
async def agent_loop(initial_message):
    messages = [initial_message]
    
    while True:
        # 1. 压缩上下文（如果太长）
        messages = compact_if_needed(messages)
        
        # 2. 构建请求
        request = {
            "messages": messages,
            "system_prompt": build_system_prompt(),
            "tools": all_available_tools()
        }
        
        # 3. 调 LLM，逐块拿返回
        response_stream = await call_llm_streaming(request)
        
        # 4. 收集 tool_use 块
        tool_calls = []
        final_text = ""
        for chunk in response_stream:
            if chunk.is_tool_use():
                tool_calls.append(chunk.tool_use)
            else:
                final_text += chunk.text
                yield chunk  # 流式输出给用户
        
        # 5a. 有工具要调 → 执行 → 把结果加回消息 → 继续循环
        if tool_calls:
            results = await execute_all_tools(tool_calls)
            messages.append(tool_calls_as_assistant_messages(tool_calls))
            messages.append(results)
            continue
        
        # 5b. 没工具了 → 返回文字 → 结束
        return final_text
```

---

## 七、今天要理解的核心问题

学完今天的内容后，你应该能回答：

1. **Agent Loop 的一次完整迭代经过哪些步骤？**
2. **消息列表（messages）在循环中是怎么变化的？**
3. **LLM 返回的 `tool_use` 块和普通文字有什么区别？**
4. **为什么需要上下文压缩（AutoCompact）？什么时候会触发？**
5. **循环的退出条件是什么？**

---

## 八、明天继续

Day 3-5 我们会深入：
- 工具是怎么注册和被调用的（Tool.ts）
- Token 预算管理是什么
- 消息是怎么构建和处理的

---

## 九、思考题（有不懂随时问我）

**Q1**: 如果用户说"帮我创建一个文件夹叫 `test`"，这个请求在 Agent Loop 里会怎么流转？

**Q2**: 为什么 Agent Loop 用 `while(true)` 而不是 `for` 循环？循环次数是固定的吗？

**Q3**: streaming 返回和非 streaming 返回，对用户感知来说有什么区别？

（答案不用交，有想法就行，不懂的来问我）
