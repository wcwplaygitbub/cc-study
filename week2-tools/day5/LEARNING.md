# Week 2 Day 5 — 自定义工具与 MCP 接入

> **核心问题**：怎么写一个自己的工具？MCP（Model Context Protocol）是什么？外部工具怎么接入 cc-haha？

---

## 一、从零写一个自定义工具

### 1.1 最小工具模板

```typescript
// src/tools/MyTool/index.ts
import { z } from 'zod/v4'
import { buildTool, type ToolDef } from '../../Tool.js'

const inputSchema = z.strictObject({
  message: z.string().describe('要处理的消息'),
})

export const MyTool = buildTool({
  name: 'my_tool',
  description: '处理特定消息的工具',
  inputSchema,
  
  async execute(params, ctx) {
    const { message } = params
    // 实际逻辑
    return {
      success: true,
      result: `处理了: ${message}`
    }
  }
} satisfies ToolDef<ReturnType<typeof inputSchema>>)
```

### 1.2 注册工具

工具写好后，需要注册才能被 Agent 使用：

```typescript
// src/tools.ts
import { MyTool } from './tools/MyTool/index.js'

export function getTools(state: AppState): Tool[] {
  return [
    MyTool,
    BashTool,
    FileReadTool,
    // ...
  ]
}
```

### 1.3 工具的完整生命周期

```
LLM 返回 tool_use
    ↓
权限检查（canUseTool）
    ↓
参数验证（validate）
    ↓
权限确认（authorize） ← 弹窗让用户确认（如果需要）
    ↓
执行（execute）
    ↓
结果格式化（mapToolResultToToolResultBlockParam）
    ↓
加入消息列表 → 继续 LLM 循环
```

### 1.4 工具的进阶能力

```typescript
export const MyTool = buildTool({
  name: 'my_tool',
  
  // UI 相关
  userFacingName: 'My Tool',           // 界面显示名
  getActivityDescription: (input) => {  // 进度描述
    return `Processing ${input.message}...`
  },
  renderToolUseMessage,                 // 工具调用时的 UI
  renderToolResultMessage,              // 结果展示的 UI
  
  // 生命周期钩子
  shouldDefer: false,                  // 是否延迟执行
  isMcp: false,                        // 是否是 MCP 工具
  
  // 输出限制
  maxResultSizeChars: 50_000,          // 最大输出字符数
  isResultTruncated: (output) => false, // 是否截断
})
```

---

## 二、MCP（Model Context Protocol）简介

### 2.1 什么是 MCP

MCP 是一种**标准化的工具接入协议**。它让外部工具提供商（如 GitHub、Slack、Google）通过统一的接口接入到任何 MCP-compatible 的 Agent 运行时。

```
┌──────────────────┐     MCP      ┌─────────────────────┐
│  Claude / cc-haha  │ ←─────────→  │  MCP Server          │
│   (MCP Client)    │              │  (GitHub, Slack...)  │
└──────────────────┘              └─────────────────────┘
```

### 2.2 MCP 核心概念

| 概念 | 说明 |
|------|------|
| **MCP Server** | 第三方服务提供的工具服务端点 |
| **MCP Client** | cc-haha 中的客户端，负责与 Server 通信 |
| **Tool** | MCP Server 暴露的工具，cc-haha 通过 MCP 调用 |
| **Resource** | MCP Server 提供的数据资源 |
| **Prompt** | 预设的 Prompt 模板 |

### 2.3 MCP Tool 的注册流程

```typescript
// src/mcpClient.ts
export class MCPClient {
  async connect(serverConfig: MCPServerConfig) {
    // 1. 建立与 MCP Server 的连接
    // 2. 获取 Server 提供的工具列表
    // 3. 把 MCP 工具转换为 cc-haha 内部格式
    // 4. 注册到工具系统
  }
}
```

---

## 三、MCPTool 实现解析

### 3.1 通用 MCP 工具

cc-haha 用一个**泛化的 MCPTool** 接受所有 MCP 工具：

```typescript
// src/tools/MCPTool/MCPTool.ts
export const MCPTool = buildTool({
  name: 'mcp',  // 会被 mcpClient 覆盖为实际工具名
  isMcp: true,  // 标记为 MCP 工具
  
  inputSchema: z.object({}).passthrough(),  // 任意输入，MCP Server 定义 schema
  
  async description() {
    return DESCRIPTION  // 会被覆盖
  },
  
  async call(params, ctx) {
    // 实际调用逻辑在 mcpClient.ts 中
    // 这里只是个代理
    const client = getMCPClient(ctx.state)
    return await client.callTool(this.mcpServerName, this.mcpToolName, params)
  }
})
```

### 3.2 为什么用 passthrough schema

MCP Server 定义的工具 schema 各不相同，cc-haha 无法提前知道有哪些字段。所以用 `z.object({}).passthrough()` 接受任意 JSON 对象，实际验证在 MCP Server 端进行。

### 3.3 MCP 工具的权限处理

```typescript
// MCPTool.ts
async checkPermissions(): Promise<PermissionResult> {
  return {
    behavior: 'passthrough',  // 直接放行，具体权限由 MCP Server 处理
    message: 'MCPTool requires permission.'
  }
}
```

---

## 四、MCP Server 配置

### 4.1 配置示例

```json
// .claude/settings.json 或 cc-haha 配置
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "xxx"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem"],
      "args": ["/allowed/path"]
    }
  }
}
```

### 4.2 MCP Server 生命周期

```typescript
// src/services/mcp/mcpClient.ts
class MCPClient {
  async start() {
    for (const [name, config] of Object.entries(servers)) {
      const process = spawn(config.command, config.args)
      // 启动 MCP Server 进程
      // 建立 stdio 通信
    }
  }
  
  async stop() {
    // 清理所有 MCP Server 进程
  }
}
```

---

## 五、自定义工具的最佳实践

### 5.1 工具 description 怎么写

```typescript
// ❌ 不好的 description
description: 'Search tool'

// ✅ 好的 description  
description: `Search the web for information. Use for:
- Looking up factual information (dates, definitions, events)
- Finding current news or weather
- Searching for technical documentation
- NOT for local file operations or code execution`
```

### 5.2 inputSchema 参数设计

```typescript
// ✅ 参数名清晰，有描述
inputSchema: z.object({
  query: z.string().describe('The search query'),
  limit: z.number().optional().describe('Max results (default 10)'),
})

// ❌ 参数名模糊，无描述
inputSchema: z.object({
  q: z.string(),
  n: z.number().optional(),
})
```

### 5.3 错误处理

```typescript
async execute(params, ctx) {
  try {
    const result = await doSomething(params)
    return result
  } catch (error) {
    return {
      error: `操作失败: ${error.message}`,
      // 可以加 errorCode 方便调试
      errorCode: 'OPERATION_FAILED'
    }
  }
}
```

### 5.4 进度展示

```typescript
async execute(params, ctx) {
  ctx.reportProgress({
    message: '正在连接服务器...',
    current: 1,
    total: 3
  })
  
  await connect()
  
  ctx.reportProgress({
    message: '获取数据...',
    current: 2,
    total: 3
  })
  
  const data = await fetchData()
  
  ctx.reportProgress({
    message: '处理结果...',
    current: 3,
    total: 3
  })
  
  return formatResult(data)
}
```

---

## 六、Week 2 整体总结

### 工具系统全景

```
Tool 定义：name + description + inputSchema
    ↓
buildTool 工厂：标准化 + 默认值填充
    ↓
工具执行：validate → authorize → execute → format
    ↓
结果返回：tool_result 消息加回消息列表
```

### 工具分类

| 类型 | 例子 | 特点 |
|------|------|------|
| **系统命令** | BashTool | 执行 shell 命令 |
| **文件操作** | FileReadTool, FileEditTool | 读写本地文件 |
| **网络工具** | WebSearchTool, WebFetchTool | 搜索和抓取网页 |
| **MCP 工具** | MCPTool | 外部工具标准化接入 |
| **业务工具** | TaskCreateTool, TeamCreateTool | cc-haha 特有能力 |

### MCP 的价值

MCP 通过标准化协议，让任何服务提供方可以一键接入 Agent 运行时，不需要修改 Agent 代码。

---

## 七、核心问题（学完应该能回答）

1. 写一个自定义工具需要哪些步骤？
2. 工具的完整生命周期是什么？
3. MCP 是什么？解决了什么问题？
4. 为什么 MCPTool 用 `passthrough` schema 而不是预定义的 schema？
5. 工具的 description 怎么写才能提高调用准确率？

---

## 八、下一步预告

Week 3 进入**记忆系统**：Agent 是怎么记住之前对话的？MEMORY.md 的机制是什么？Session 之间如何共享记忆？