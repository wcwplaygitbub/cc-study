# Week 2 Day 5 学习总结

> 学习日期：2026-05-18
> 学习时长：约 1 小时
> 理解程度：☑ 大部分理解

---

## 一、核心理解

### 自定义工具四步曲

```typescript
// 1. 定义 schema
const inputSchema = z.object({ ... })

// 2. buildTool 创建
export const MyTool = buildTool({
  name: 'my_tool',
  description: '...',
  inputSchema,
  async execute(params, ctx) { ... }
})

// 3. 注册到 getTools
export function getTools(state): Tool[] {
  return [MyTool, ...]
}

// 4. 使用
// LLM 会自动在 description 中匹配到 'my_tool' 并调用
```

### MCP 价值

MCP 是外部工具接入的"USB-C"——不管什么厂商，只要实现了 MCP Server，就能即插即用到任何 MCP Client（cc-haha）。

### 工具 description 最佳实践

description 写清楚：
1. 这个工具**用来做什么**（核心用途）
2. 什么场景**应该用**它
3. 什么场景**不该用**（避免误调用）
4. 返回什么（可选）

---

## 二、签字确认

Week 2 完成：☑ 是
继续 Week 3（记忆系统）：☑ 是