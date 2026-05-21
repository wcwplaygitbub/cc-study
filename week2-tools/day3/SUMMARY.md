# Week 2 Day 3-5 学习总结

> 学习日期：2026-05-18
> 学习时长：约 2.5 小时
> 理解程度：□ 完全理解  □ 大部分理解  □ 部分理解  □ 需要重学

---

## 一、当天问题记录

### Q1: FileReadTool 读取大文件时怎么避免超出 token 限制？

**A:** FileReadTool 通过**分段读取 + token 预算控制**处理大文件：
- 支持 `offset` + `limit` 参数，只读取需要的部分
- 对 PDF 限制最多一次读 10 页（`PDF_MAX_PAGES_PER_READ`）
- 超过 3000 token 的 PDF 只保留引用而不是全文（`PDF_AT_MENTION_INLINE_THRESHOLD`）
- 返回文件总行数，让 LLM 知道还有多少内容未读

### Q2: FileEditTool 的 old_string 匹配和 sed 有什么区别？

**A:** FileEditTool 用**精确字符串匹配**，sed 用正则表达式。

| | FileEditTool | sed |
|---|---|---|
| 匹配方式 | exact string | regex |
| 替换策略 | 单次或全局（`replace_all`） | 全局 |
| 并发保护 | ✅ mtime staleness check | ❌ 无 |
| 原子性 | 高（先读后写） | 低（直接改） |

### Q3: staleness check 是怎么防止并发修改冲突的？

**A:** LLM 读取文件时会记录 `fileMtime`（文件修改时间）。编辑时如果发现当前文件的 mtime 与之前记录的不同，说明文件在 Agent 读取之后、编辑之前被外部修改了，此时拒绝执行，返回 `FILE_UNEXPECTEDLY_MODIFIED_ERROR`，让 LLM 重新读取再尝试。

### Q4: WebSearchTool 和 WebFetchTool 的分工是什么？

**A:** 
- **WebSearchTool**：搜索关键字，返回相关网页的 `[title, url]` 列表
- **WebFetchTool**：抓取指定 URL，执行 prompt 提取页面中的特定信息

两者配合：Search 找相关页面 → Fetch 抓取详细内容 → LLM 阅读理解后回答。

### Q5: MCP 是什么？为什么需要它？

**A:** MCP = Model Context Protocol，一种标准化的工具接入协议。它让外部服务（GitHub、Slack、Google Calendar 等）通过统一的 Server-Client 接口接入 Agent，不需要每个 Agent 运行时单独适配。用 `z.object({}).passthrough()` schema 因为 MCP Server 自己定义工具 schema，cc-haha 无法预知字段。

---

## 二、核心理解

### 文件工具架构

```
权限检查（filesystem permissions）
    ↓
路径验证（path validation）
    ↓
文件操作（read/write/edit）
    ↓
结果格式化（UI 渲染 + git diff）
```

### 网络工具的 fallback 设计

优先用 Anthropic native web search → 出错时自动 fallback 到第三方 API → 保证搜索能力不中断。

### 工具注册到生效的流程

工具定义 → buildTool 标准化 → getTools() 收集 → 传给 LLM 的 tools 参数 → LLM 调用 → canUseTool 检查 → execute → 结果加回消息列表

---

## 三、收获与疑问

### 今天最大的收获

工具系统的设计很模块化：每个工具都实现统一的 Tool 接口，通过 buildTool 工厂标准化。这意味着只要理解了一个工具的实现，其他工具基本类似。同时 MCP 的设计让我看到生态扩展的标准方式。

### 还需要深入理解的地方

MCP Server 的具体通信协议细节（stdio + JSON-RPC 的具体帧格式），以及 LocalShellTask 的子进程管理和生命周期。

### 下一步学习计划

Week 3 记忆系统：理解 Agent 的持久化记忆机制、Session 之间如何共享上下文。