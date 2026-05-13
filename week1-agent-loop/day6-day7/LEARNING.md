# Week 1 Day 6-7 — Agent Loop 串联 + Week 1 复盘

> **目标**：把前几天的知识串起来，理解完整的 Agent Loop 闭环，并形成自己的架构认知

---

## 一、Agent Loop 完整闭环

### 1.1 闭环全流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                          用户发送消息                              │
│                    "帮我创建一个文件夹 test"                       │
└────────────────────────────┬────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  阶段1：请求构建                                                  │
│  · 消息列表 = [系统提示] + [记忆] + [对话历史]                      │
│  · Token 预算检查（超过 80% → 压缩）                               │
│  · 工具列表 = 所有可用工具的定义                                    │
└────────────────────────────┬────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  阶段2：LLM 调用（Streaming）                                      │
│  · 流式返回 text / thinking / tool_use 块                         │
│  · text → 实时展示给用户                                          │
│  · tool_use → 收集到列表                                          │
└────────────────────────────┬────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              │                               │
              ▼                               ▼
    ┌──────────────────┐           ┌─────────────────────────┐
    │ 没有 tool_use     │           │ 有 tool_use（1个或多个）   │
    │ → 输出文字回复    │           │ → 执行工具               │
    │ → 循环结束        │           │ → 结果加回消息列表       │
    └──────────────────┘           │ → 继续循环               │
                                     └─────────────────────────┘
```

### 1.2 关键模块及其职责

| 模块 | 文件位置 | 职责 | 在循环中的位置 |
|------|---------|------|--------------|
| 消息构建 | `query.ts` | 把系统提示/记忆/历史消息组织成请求 | 每次循环第一步 |
| Token管理 | `bootstrap/state.ts` | 检查预算，可能触发压缩 | 请求构建前 |
| AutoCompact | `services/compact/` | 压缩超长上下文 | Token超限时 |
| LLM调用 | `deps.callModel()` | 发送请求，Streaming接收 | 循环第二步 |
| 工具执行 | `toolOrchestration.ts` | 执行 tool_use，调工具 | tool_use 收集完后 |
| 结果处理 | `query.ts` | 把工具结果转成消息加回列表 | 工具执行完后 |

---

## 二、Agent Loop vs 传统程序

### 2.1 传统程序

```
输入 → 计算 → 输出 → 结束
```

### 2.2 Agent 程序

```
输入 → 思考（LLM）→ 决定行动（可能调用工具）
                        ↓
                   行动结果 → 继续思考（LLM）
                                      ↓
                                 决定行动（可能再调工具）
                                      ↓
                                  ... 循环 ...
                                      ↓
                                 文字回复 → 结束
```

**本质区别**：Agent Loop 的循环次数是**不固定的**，取决于 LLM 的决策。简单问题一轮搞定，复杂问题可能循环几十轮。

---

## 三、设计模式总结

### 3.1 Generator（生成器）模式

```typescript
// query.ts 使用 async generator
async function* query(params) {
  // yield 用来输出"事件"（流式文字、工具结果、状态更新）
  // 每 yield 一次，用户能看到一次输出
  
  yield "正在思考..."  // 用户看到
  yield "好的，我来..."  // 用户又看到
  
  // 内部可以 yield 很多次
  // 外部 for await 遍历这些输出
}
```

**为什么用 Generator？**
- 流式输出：用户不用等全部生成完
- 事件驱动：工具结果、错误状态都可以作为事件 yield 出来
- 可控性强：外部可以控制循环的暂停/继续

### 3.2 消息队列模式

消息列表（`messages[]`）是一个**队列**，每次循环：
- 右边不断追加（用户消息、AI回复、工具结果）
- 左边可能被压缩（AutoCompact 把旧消息变成摘要）

### 3.3 责任链模式

一次工具调用经过：
```
LLM决定 → 权限检查 → 参数验证 → 执行 → 结果格式化 → 错误处理
```

每一步由不同模块负责，像链条一样串起来。

---

## 四、最小 Agent Loop 伪代码

理解 cc-haha 的 Agent Loop 后，你可以这样设计自己的：

```python
import asyncio

class SimpleAgentLoop:
    def __init__(self, llm_client, tools):
        self.llm = llm_client
        self.tools = {t.name: t for t in tools}
        self.messages = []
    
    async def run(self, user_input):
        # 1. 加用户消息
        self.messages.append({"role": "user", "content": user_input})
        
        # 2. 循环
        while True:
            # 2a. 构建请求
            request = {
                "model": self.llm.model,
                "messages": self.messages,
                "tools": [t.to_schema() for t in self.tools.values()]
            }
            
            # 2b. 调用 LLM（streaming）
            response = ""
            tool_calls = []
            
            async for chunk in self.llm.stream(request):
                if chunk.type == "text":
                    response += chunk.text
                    print(chunk.text, end="", flush=True)
                elif chunk.type == "tool_use":
                    tool_calls.append(chunk.data)
            
            # 2c. 有工具要调？
            if not tool_calls:
                break  # 没有工具，结束
            
            # 2d. 执行工具
            results = []
            for tc in tool_calls:
                tool = self.tools[tc.name]
                result = await tool.execute(tc.arguments)
                results.append({
                    "role": "tool",
                    "tool_call_id": tc.id,
                    "content": str(result)
                })
            
            # 2e. 工具结果加回消息列表
            self.messages.append({"role": "assistant", "content": response})
            self.messages.append({"role": "tool", "content": results})
        
        return response


# 使用示例
tools = [BashTool(), FileReadTool(), FileEditTool()]
agent = SimpleAgentLoop(llm=anthropic_client, tools=tools)
await agent.run("帮我创建一个文件夹叫 test")
```

---

## 五、Week 1 知识点全景图

```
Agent Loop 核心循环（query.ts）
    │
    ├── 消息构建
    │   ├── 系统提示（固定不变）
    │   ├── 记忆注入（MEMORY.md / 主题文件）
    │   ├── 对话历史（messages 列表）
    │   └── Skill 上下文（当前激活的 Skill）
    │
    ├── Token 管理
    │   ├── 预算检查（超过 80% 触发压缩）
    │   ├── AutoCompact（压缩成摘要）
    │   └── 微压缩（microcompact，减少冗余）
    │
    ├── LLM 调用
    │   ├── Streaming 流式返回
    │   ├── 文本 / 思考 / 工具块 分流
    │   └── Fallback 模型降级
    │
    ├── 工具执行
    │   ├── 工具注册（Tool.ts）
    │   ├── 工具分发（toolOrchestration.ts）
    │   ├── 权限检查（canUseTool）
    │   └── 结果格式化（tool_result）
    │
    └── 循环控制
        ├── 有 tool_use → continue
        └── 无 tool_use → return 完成
```

---

## 六、Week 1 复盘问题

学完 Week 1 后，回答这些问题，检验理解程度：

### 问题 1
如果用户说"帮我看看当前目录下有什么文件"，请描述这个消息在 Agent Loop 中从收到到最终回复的完整过程。

___



### 问题 2
假设一个对话进行了 50 轮，消息列表越来越长。请描述：
- 什么时候会触发压缩？
- 压缩后消息列表变成什么样？
- 压缩后 LLM 还能知道之前的对话内容吗？

___



### 问题 3
在 cc-haha 中，`src/tools/AgentTool/` 是用来做什么的？它和其他工具（如 BashTool）有什么本质区别？

___



### 问题 4
尝试用自己的话解释：为什么 Agent Loop 用 while(true) 而不是固定次数的循环？

___



---

## 七、下一步预告

**Week 2：工具系统**

深入研究 cc-haha 中的工具是怎么实现的：
- BashTool：怎么执行命令、怎么捕获输出
- FileEditTool：怎么实现 diff 展示
- MCPTool：外部工具怎么接入
- 怎么写一个自定义工具

---

## 八、Week 1 最终总结（学完后填写）

> 学完 Week 1 后，在这里写下你对 Agent Loop 的完整理解

### Agent Loop 本质

___



### 我理解了的核心流程

___



### 我觉得最巧妙的设计

___



### 我还有疑问的地方

___



### 对我后续开发的意义

___

