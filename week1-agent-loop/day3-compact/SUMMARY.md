# Week 1 Day 3 学习总结

> 学习日期：2026-05-14（问题解答：2026-05-15）  
> 学习时长：约 30 分钟  
> 理解程度：□ 完全理解  ■ 大部分理解  □ 部分理解  □ 需要重学

---

## 一、当天问题记录

### Q1: AutoCompact 触发的时候，是压缩所有消息还是只压缩一部分？

**A**: 只压缩 compact boundary 之前的历史消息。boundary 之后的最近对话不压缩，确保重要上下文不被丢弃。

---

### Q2: 压缩后的摘要消息是什么角色？为什么？

**A**: 是一个新的假的 assistant 消息，内容是压缩后的摘要文本。这样 LLM 看到这条消息时，会把它当作一条正常的 AI 回复，从而理解整个对话的历史背景。

---

### Q3: 压缩摘要具体是怎么生成的？是用同一个 LLM 模型还是专门的压缩模型？

**A**: 用**同一个 LLM 模型**生成，不是专门的压缩模型。

核心流程：
1. 构建压缩 Prompt（来自 `prompt.js` 的 `getCompactPrompt()`），系统提示是 `"You are a helpful AI assistant tasked with summarizing conversations."`
2. 调用 `queryModelWithStreaming`，使用和主循环同一个模型（`context.options.mainLoopModel`）
3. 工具限制：压缩时只有 `FileReadTool` 可用，其他工具全部拒绝（`createCompactCanUseTool` 返回 `deny`）
4. Thinking 关闭：`thinkingConfig: { type: 'disabled' }`，不浪费 tokens 在思考上
5. 两种路径：
   - **Cache sharing（优先）**：尝试复用主对话的 prompt cache，通过 `runForkedAgent` 在 forked agent 里执行
   - **Streaming fallback（兜底）**：如果 cache sharing 失败，走普通流式调用

---

### Q4: 摘要的质量怎么保证？

**A**: 通过多层机制保证：

| 机制 | 说明 |
|------|------|
| **Circuit Breaker** | 连续 3 次压缩失败后不再重试，避免浪费 API calls |
| **PTL Retry** | 如果压缩请求本身触发 `prompt_too_long`，会截断头部消息重试（最多3次） |
| **Image Stripping** | 图片/文档在发送前会被替换成 `[image]` / `[document]` 标记，避免压爆自己 |
| **Token Budget** | 多个预算限制：摘要最多 20K output、文件恢复最多 50K、每个 skill 最多 5K |
| **结果校验** | 检查是否以 `PROMPT_TOO_LONG_ERROR_MESSAGE` 开头或 API error prefix，是则走重试逻辑 |
| **Post-compact Hooks** | 压缩后重新注入文件附件、工具 delta、skill 信息，保证上下文不断 |

---

### Q5: 压缩后消息列表具体是什么结构？

**A**: 压缩后消息列表结构：

```
[
  boundaryMarker,     ← SystemCompactBoundaryMessage，标记压缩发生
  summaryMessage,      ← User message，内容是摘要文本，isCompactSummary=true，isVisibleInTranscriptOnly=true 对用户不可见
  ...messagesToKeep,   ← 保留的消息（通常是最近的对话）
  ...attachments,      ← 文件、技能、计划等附件重新注入
  ...hookResults       ← session start hooks 结果
]
```

---

## 二、核心理解

### 触发条件：两步检查

1. **Token 预算检查**：每次循环前计算消息列表总 tokens，超过 context window 的 80% 就触发
2. **Compact Boundary 保护**：boundary 之前的消息才压缩，之后的保留

___



### 压缩过程

1. 选中 boundary 前的消息块
2. 用 LLM 生成压缩摘要（约 1K tokens）
3. 把原始消息替换成摘要消息
4. 继续正常循环

___



### 压缩后的消息列表结构

```
压缩后：
[
  summaryMessage,   // 压缩摘要（假 assistant 消息）
  msg48, msg49, msg50   // boundary 之后的关键消息保留
]
```

___



## 三、收获与疑问

### 今天最大的收获

理解了 AutoCompact 不是简单"删除旧消息"，而是用 LLM 生成摘要来保留关键信息，本质是用信息密度换 token 空间。这是一个很巧妙的设计，在有限的上下文窗口内尽可能保留对话历史。

___



### 还需要深入理解的地方

（已解答）压缩摘要具体是怎么生成的？是用同一个 LLM 模型还是专门的压缩模型？摘要的质量怎么保证？

（已解答）压缩后消息列表具体是什么结构？

___



### 下一步学习计划

Day 4-5 继续工具系统，学习工具注册机制、工具执行流程、Token 预算检查等。
