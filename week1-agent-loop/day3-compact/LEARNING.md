# Week 1 Day 3 — 动态压缩（AutoCompact）

> **核心问题**：当对话越来越长，消息列表越来越膨胀，Agent 是怎么在快要爆之前主动"瘦身"，释放上下文空间的？

---

## 一、为什么需要动态压缩

LLM 有上下文窗口限制，比如 100K tokens。对话过程中消息列表会不断膨胀：

```
对话开始：1K tokens         ✅
第10轮对话：20K tokens      ✅
第50轮对话：95K tokens       ⚠️ 快触顶了
第51轮对话：105K tokens      ❌ 超了，放不下了
```

如果在第 50 轮时主动压缩，把旧消息变成摘要，就能继续对话。这就是 **AutoCompact（自动压缩）**。

---

## 二、触发机制：两步检查

### 第一步：Token 预算检查

每次循环开始前，会计算当前消息列表的总 token 数：

```typescript
const totalTokens = countTokens(messages)
const limit = model.contextWindow        // e.g., 100K tokens
const threshold = limit * 0.8            // 80K，安全阈值（80%）

if (totalTokens > threshold) {
  // 触发压缩
  triggerAutoCompact()
}
```

用 **80%** 而不是 100%——留 20% 缓冲，因为 LLM 回复本身也会产生新 tokens。

### 第二步：Compact Boundary（压缩边界）

不是所有消息都能被压缩。cc-haha 有一个 **compact boundary** 机制，保护最近的关键对话不被压缩：

```typescript
// 伪代码
const preserveable = messages.slice(compactBoundary)   // boundary 之后的，保留
const compressible = messages.slice(0, compactBoundary) // boundary 之前的，可以压
```

boundary 通常设在最近 N 条消息之后，确保压缩不影响最近对话的完整性。

---

## 三、压缩过程：具体怎么压

### 步骤 1 — 选择要压缩的消息块

把 **compact boundary 之前** 的消息选中，这可能是一大段历史对话：

```
可压缩区域：
[用户: 帮我写排序]           → 1K tokens
[助手: 好的，快速排序...]     → 3K tokens
[用户: 加个二分查找]         → 0.5K tokens
[助手: 好了，加上...]        → 4K tokens
[用户: 测试一下]            → 0.5K tokens
... (50轮对话，共 85K tokens)
```

### 步骤 2 — 生成压缩摘要

用 LLM 本身把这段消息压缩成一段摘要：

```
原始 (85K tokens):
[50轮对话，关于实现排序算法、增加二分查找、测试、调Bug...]

压缩后 (~1K tokens):
"[压缩摘要] 用户要求实现排序算法，助手写了快速排序；
  后续48轮对话主要围绕：增加二分查找、修复边界Bug、测试用例补充，
  最终完成了带二分查找的排序工具，输出验证通过。"
```

### 步骤 3 — 替换消息列表

```typescript
// 压缩前
messages = [
  msg1, msg2, msg3, ...msg50   // 50条，85K tokens
]

// 压缩后
messages = [
  summaryMessage,               // 压缩后的摘要（新的假消息）
  msg48, msg49, msg50           // boundary 之后的关键消息保留
]
// 总计：约 10K tokens ✅
```

注意：压缩后的摘要是一个**新的假的 assistant 消息**，内容是摘要文本，插入消息列表最前面（被压缩区的起始位置）。

---

## 四、压缩后的循环怎么继续

```typescript
while (true) {
  // 1. 计算 tokens
  const tokens = countTokens(messages)
  
  if (tokens > threshold) {
    // 2. 执行压缩
    const { compressedMessages, summary } = await autoCompact(messages)
    messages = compressedMessages
    // 3. 摘要消息插入消息列表
    messages.unshift(summaryMessage)
  }
  
  // 4. 继续正常循环
  response = await callLLM(messages)
  // ...
}
```

---

## 五、AutoCompact 在 cc-haha 源码中的位置

| 文件 | 作用 |
|------|------|
| `src/services/compact/autoCompact.ts` | 压缩逻辑核心 |
| `src/bootstrap/state.ts` | Token 预算管理 |
| `src/query.ts` | 循环中调用压缩的入口 |

---

## 六、关键设计点

| 设计 | 说明 |
|------|------|
| **边界保护** | 最近几轮不压缩，防止重要上下文丢失 |
| **摘要生成** | 用 LLM 自己生成摘要，保证信息完整 |
| **不可逆** | 压缩是单向的，原始消息被丢弃，无法还原 |
| **静默进行** | 压缩对用户透明，不影响对话体验 |
| **多次触发** | 对话足够长时，可能触发多次压缩 |

---

## 七、直观比喻

把消息列表想象成一张桌子：

- 桌上文件越来越多 → 越来越挤 → 快放不下了
- 这时候你把**桌角的文件**（旧消息）收到**文件夹**（摘要）里
- 桌上只留最近的文件，腾出空间继续用
- 文件夹里有压缩内容，下次翻的时候还能看到大概是什么

---

## 八、核心问题（学完应该能回答）

1. **AutoCompact 的触发条件是什么？什么时候会触发？**
2. **Compact Boundary 是怎么保护关键上下文的？**
3. **压缩后的摘要消息在消息列表里是什么角色？**
4. **压缩是单向的还是可逆的？为什么这样设计？**

---

## 九、下一步

Day 4-5 我们继续深入 **工具系统**：Agent 是怎么让 LLM 知道有哪些工具可用的？BashTool、FileEditTool 是怎么实现的？
