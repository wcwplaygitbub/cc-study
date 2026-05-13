# Week 1 Day 3-5 — 工具系统与 Token 管理

> **核心问题**：Agent 是怎么让 LLM 知道有哪些工具可用的？工具调用的结果怎么回到循环？Token 预算怎么影响循环？

---

## 一、工具注册机制（Tool.ts）

### 1.1 什么是工具定义？

LLM 本身不知道能做什么——需要告诉它"你有哪些工具可以调用"。每个工具有一个定义：

```typescript
// 工具定义的通用结构（伪代码）
{
  name: "Bash",                    // 工具名称，LLM 用这个名字来调用
  description: "执行Shell命令",    // 描述，告诉 LLM 什么场景用它
  input_schema: {                  // 参数规范，LLM 知道怎么传参
    type: "object",
    properties: {
      command: { type: "string" }
    },
    required: ["command"]
  },
  // 实际执行函数
  async execute(args) {
    return await runBash(args.command)
  }
}
```

### 1.2 工具怎么注册到 Agent

```typescript
// Tool.ts 中的核心逻辑（伪代码）
class ToolRegistry {
  private tools: Map<string, Tool> = new Map()
  
  // 注册一个工具
  register(tool: Tool) {
    this.tools.set(tool.name, tool)
  }
  
  // 根据名字找工具
  findByName(name: string): Tool | undefined {
    return this.tools.get(name)
  }
  
  // 把所有工具转成 LLM 认识的格式（tool_use schema）
  toLLMTools() {
    return Array.from(this.tools.values()).map(t => ({
      name: t.name,
      description: t.description,
      input_schema: t.input_schema
    }))
  }
}
```

### 1.3 cc-haha 中的工具目录

```
src/tools/
├── BashTool/          # 执行命令行
├── FileReadTool/       # 读取文件
├── FileEditTool/       # 编辑文件（diff 展示）
├── GlobTool/           # 文件模式匹配
├── GrepTool/           # 文本搜索
├── AgentTool/          # spawn 子Agent ← 重要！
├── MCPTool/            # 外部 MCP 工具接入
└── ... (共 30+ 个工具)
```

---

## 二、工具执行（toolOrchestration.ts）

### 2.1 工具执行的完整流程

当 LLM 返回 `tool_use` 块后，Agent 执行以下流程：

```
LLM 返回 tool_use 块
    │
    ▼
解析工具名和参数
    │
    ▼
权限检查：这个工具当前会话有没有权限用？
    │
    ├─ 有权限 ──→ 执行工具
    └─ 没权限 ──→ 暂停，等待用户授权
                  ├─ 用户授权 ──→ 执行工具
                  └─ 用户拒绝 ──→ 返回错误给 LLM
    │
    ▼
执行工具（同步或异步）
    │
    ▼
把执行结果格式化成 tool_result
    │
    ▼
tool_result 加回消息列表 → 继续发给 LLM → 下一轮循环
```

### 2.2 权限检查模式

cc-haha 支持多种权限模式（取决于用户配置）：

| 模式 | 行为 |
|------|------|
| `allow` | 所有工具直接执行，不问 |
| `bringYourOwn` | 每个工具第一次用时问一次 |
| `manual` | 每个工具每次都用时问 |
| `plan` | Plan 模式下部分工具可用 |

### 2.3 工具执行伪代码

```typescript
async function runTools(toolCalls: ToolUseBlock[], context: ToolContext) {
  const results = []
  
  for (const toolCall of toolCalls) {
    const tool = findToolByName(context.tools, toolCall.name)
    
    if (!tool) {
      results.push({ error: `未知工具: ${toolCall.name}`, tool_use_id: toolCall.id })
      continue
    }
    
    // 权限检查
    if (!context.canUseTool(tool.name)) {
      // 需要用户授权，暂停等待
      await waitForUserPermission(tool.name)
    }
    
    try {
      const result = await tool.execute(toolCall.input, context)
      results.push({ result, tool_use_id: toolCall.id })
    } catch (error) {
      results.push({ error: error.message, tool_use_id: toolCall.id })
    }
  }
  
  return results
}
```

---

## 三、Token 预算管理

### 3.1 什么是 Token？

LLM 不是按字符处理的，是按 **Token**（词元）处理的。一个 Token 大约是：
- 英文：4个字符 ≈ 1个Token
- 中文：1-2个汉字 ≈ 1个Token

LLM 有上下文窗口限制（如 100K tokens），超过就处理不了。

### 3.2 Token 预算检查流程

```
每次发请求给 LLM 前：
    │
    ▼
计算当前 messages 的总 token 数
    │
    ▼
是否超过安全阈值（如 80%）？
    │
    ├─ 是 ──→ 触发 AutoCompact（压缩旧消息）
    │          └─→ 用压缩后的摘要替换
    │
    └─ 否 ──→ 正常发送
```

### 3.3 AutoCompact 压缩策略

当消息太长时，Claude Code 会把一段消息压缩成一段摘要：

```
原始消息（100条，共 90K tokens）：
[
  "用户：帮我写排序算法",
  "助手：好的，这是快速排序...",
  "用户：能不能加个二分查找？",
  "助手：好的，加上...",
  ... (100条)
]

压缩后：
[
  "用户：帮我写排序算法",
  "助手：好的，这是快速排序...",
  "[压缩摘要] 后续48轮对话，主题：排序算法优化，增加了二分查找...",
]
```

压缩后 token 大幅减少，但保留了关键信息。

---

## 四、消息构建详解

### 4.1 消息是怎么组成的

每次发给 LLM 的请求，消息列表由多部分组成：

```typescript
function buildRequest(messages, context) {
  return {
    messages: [
      // 1. 系统提示（不变，每个请求都带）
      { role: 'system', content: systemPrompt },
      
      // 2. 记忆内容（从 MEMORY.md 注入）
      { role: 'system', content: memoryContent },
      
      // 3. 对话历史
      ...messages,
      
      // 4. Skill 上下文（当前激活的 Skill）
      { role: 'system', content: skillContext },
    ],
    tools: context.tools  // 所有可用工具的定义
  }
}
```

### 4.2 工具结果怎么回到循环

```
工具执行完成后：
    │
    ▼
生成 tool_result 消息块
{
  role: 'user',  // ← 注意是 user 角色！
  content: [
    {
      type: 'tool_result',
      tool_use_id: 'toolu_01XXXX',
      content: '文件已创建，行数：25'
    }
  ]
}
    │
    ▼
加到 messages 列表
messages.push(toolResultMessage)
    │
    ▼
继续循环 → 发给 LLM → LLM 看到结果 → 做下一步决策
```

**关键**：工具结果是作为 `user` 角色的消息加回去的，这样 LLM 就知道"我之前做了什么，结果是什么"。

---

## 五、Streaming 流式输出

### 5.1 什么是 Streaming

普通 API：等 LLM 生成完所有文字，再一次性返回（等待时间长）
Streaming：LLM 生成一个字，就立即返回一个字（响应快，能看到思考过程）

```typescript
// 非 Streaming（等全部生成完）
response = await callLLM(request)
console.log(response.text)  // 一次性打印完整回复

// Streaming（边生成边返回）
for await (const chunk of callLLMStreaming(request)) {
  if (chunk.type === 'text') {
    print(chunk.text, end='')  // 一个字一个字打印
  }
}
```

### 5.2 Streaming 怎么和工具调用共存

```typescript
for await (const chunk of responseStream) {
  if (chunk.type === 'text' || chunk.type === 'thinking') {
    yield chunk  // 立即展示给用户
  }
  
  if (chunk.type === 'tool_use') {
    toolUseBlocks.push(chunk)  // 收集起来，后面统一执行
  }
}

// 所有 chunks 都收完了，才执行工具
if (toolUseBlocks.length > 0) {
  const results = await executeTools(toolUseBlocks)
  // ...继续循环
}
```

---

## 六、伪代码：完整的 Agent Loop

```python
async def agent_loop(user_message):
    messages = [user_message]
    
    while True:
        # 1. 检查 token 预算，可能压缩
        if token_count(messages) > TOKEN_LIMIT * 0.8:
            messages = compact(messages)
        
        # 2. 构建请求
        request = {
            "messages": messages,
            "system_prompt": build_system_prompt(),
            "tools": get_all_tool_definitions()
        }
        
        # 3. 流式调用 LLM
        tool_calls = []
        text_output = ""
        
        async for chunk in call_llm_streaming(request):
            if chunk.is_text():
                text_output += chunk.text
                yield chunk  # 实时展示给用户
            elif chunk.is_thinking():
                yield chunk  # 思考过程也展示
            elif chunk.is_tool_use():
                tool_calls.append(chunk.tool_use)
        
        # 4. 判断：是否有工具要执行
        if tool_calls:
            # 权限检查
            for tc in tool_calls:
                if not user_allows_tool(tc.name):
                    await user.confirm(tc.name)
            
            # 执行所有工具
            results = []
            for tc in tool_calls:
                result = await execute_tool(tc.name, tc.input)
                results.append(format_tool_result(tc.id, result))
            
            # 工具结果作为 user 消息加回去
            messages.append(assistant_message(text_output, tool_calls))
            messages.append(user_message(results))
            # 继续循环
            continue
        else:
            # 结束
            return text_output
```

---

## 七、核心问题（学完这几天应该能回答）

1. **工具定义包含哪几个关键字段？LLM 是怎么通过这些字段知道怎么调用工具的？**

2. **工具执行结果为什么要作为 `role: 'user'` 的消息加回消息列表，而不是其他角色？**

3. **AutoCompact 触发后，消息列表是怎么变化的？压缩后的摘要能完全替代原始消息吗？**

4. **Streaming 和非 Streaming 两种模式下，tool_use 块是什么时候被收集的？**

5. **如果用户连续说了3句话，消息列表里会有几条消息？它们是怎么排列的？**

---

## 八、明天继续

Day 6-7：
- 把 Agent Loop 的各部分串起来
- 写最小可运行的 Demo 伪代码
- Week 1 复盘总结
