# Week 2 Day 3-5 学习总结

> 学习日期：2026-05-18
> 学习时长：约 3 小时（Day 3-5 合并）
> 理解程度：☑ 大部分理解，部分细节需实践巩固

---

## 一、工具系统全景

### 工具分类

| 类型 | 工具 | 核心职责 |
|------|------|---------|
| 系统命令 | BashTool | 执行 shell 命令 |
| 文件读取 | FileReadTool | 读取本地文件，支持图片/PDF/分段 |
| 文件编辑 | FileEditTool | 精确字符串替换，含 staleness check |
| 网页搜索 | WebSearchTool | 多后端 fallback 的网页搜索 |
| 网页抓取 | WebFetchTool | URL 内容提取 + prompt 处理 |
| 外部协议 | MCPTool | 标准化 MCP 协议接入 |

### 工具接口三要素

```typescript
interface Tool {
  name: string           // 唯一标识
  description: string   // LLM 靠它匹配场景
  inputSchema: z.ZodType // 参数定义
}
```

### 工具执行链路

```
canUseTool (权限检查)
  → validate (参数验证)
  → authorize (用户确认)
  → execute (实际执行)
  → mapToolResultToToolResultBlockParam (格式化)
  → 加回消息列表
```

---

## 二、关键设计点

### FileEditTool 的 staleness check

LLM 读取文件时记录 mtime，编辑前对比当前 mtime。若不同说明文件被外部修改，拒绝执行，让 LLM 重新读取。

### WebSearchTool fallback

```
原生搜索 → 失败/不可用
  → 第三方 API fallback
  → 保证搜索不中断
```

### MCP passthrough schema

MCP Server 定义的 schema 各不同，cc-haha 用 `z.object({}).passthrough()` 接受任意输入，验证交给 MCP Server 自己做。

---

## 三、自我评估

| 知识点 | 理解程度 | 备注 |
|--------|---------|------|
| 工具接口三要素 | ☑ 完全理解 | |
| BashTool 安全检查 | ☑ 大部分理解 | 具体规则还需看代码 |
| FileEditTool staleness | ☑ 完全理解 | 很有用的设计 |
| WebSearchTool fallback | ☑ 大部分理解 | 需看具体 API 对接 |
| MCP 协议 | ☑ 大部分理解 | 协议细节需深入 |
| 自定义工具 | ☑ 大部分理解 | 实际写一个会更清晰 |

---

## 四、Week 2 最终总结

### Agent Loop + 工具系统 = 完整能力

```
用户输入
  ↓
Agent Loop（消息构建 → LLM 推理）
  ↓
    ├→ 文本回复 → 返回用户
    └→ 工具调用 → 权限检查 → execute → 结果加回消息列表
                    ↓
               继续 LLM 推理（循环）
```

Week 1 理解了 Agent 的循环机制，Week 2 理解了 Agent 的行动能力。两者结合，Agent 才能真正"帮你做事"而不是只会说话。

---

## 五、签字确认

Week 2 完成：☑ 是
继续 Week 3（记忆系统）：☑ 是