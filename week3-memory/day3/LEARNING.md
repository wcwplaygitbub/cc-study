# Week 3 Day 3 — 记忆加载与 Daily Log 模式

> **核心问题**：记忆是什么时候加载进对话的？AutoMemory 的 daily log 模式是什么？Session 之间怎么共享记忆？

---

## 一、记忆加载时机

### 1.1 加载位置：system prompt 构建时

记忆不是"对话过程中按需加载"的，而是在**对话一开始**就注入到 system prompt 里：

```typescript
// src/bootstrap/state.ts
// 或 src/systemPrompt.ts
async function buildSystemPrompt() {
  // ...
  const memoryPrompt = await loadMemoryPrompt()
  if (memoryPrompt) {
    sections.push({ type: 'memory', content: memoryPrompt })
  }
  // ...
}
```

### 1.2 加载内容

`loadMemoryPrompt()` 返回的内容包含：

1. **记忆行为指导**（怎么写记忆、什么时候写）
2. **MEMORY.md 索引内容**（所有记忆文件的入口列表）

```typescript
// 实际注入进 system prompt 的内容
`# auto memory

You have a persistent, file-based memory system at ~/.claude/projects/xxx/memory/

## Types of memory
[4种记忆类型的说明...]

## What NOT to save in memory
[不应该保存的内容...]

## How to save memories
[如何保存记忆...]

## MEMORY.md
- [user_role](user/user_role.md) — Senior engineer, prefers detailed explanations
- [feedback_no_mocks](feedback/feedback_no_mocks.md) — Use real DB in tests`
```

### 1.3 截断机制

如果 MEMORY.md 太大（超过 200 行或 25KB），会被截断：

```typescript
// src/memdir/memdir.ts
const MAX_ENTRYPOINT_LINES = 200
const MAX_ENTRYPOINT_BYTES = 25_000

export function truncateEntrypointContent(raw: string): EntrypointTruncation {
  // 1. 先按行截断（最多 200 行）
  // 2. 再按字节截断（最多 25KB，在最后一个换行符处切断）
  // 返回截断内容 + 警告信息
}
```

这保证了每次加载 MEMORY.md 的 token 成本是可控的。

---

## 二、两种记忆模式：Standard vs Daily Log

### 2.1 Standard 模式（默认）

适用于 Claude Code 等**会话式 Agent**：

- 记忆存在 `memory/MEMORY.md` + 各类型子目录
- Agent 直接读写记忆文件
- **每次写新记忆后要更新 MEMORY.md 索引**

```
memory/
├── MEMORY.md           ← 索引（每次写新记忆后更新）
├── user/
│   └── user_role.md
├── feedback/
│   └── feedback_preference.md
└── project/
    └── project_deadline.md
```

### 2.2 Daily Log 模式（KAIROS feature）

适用于**长期助手（Assistant Mode）**的 append-only 日志：

- 记忆以**时间戳的形式**追加到当天的日志文件
- **不直接更新 MEMORY.md**（索引由夜间 process 维护）
- Agent 永远只 append，不重写、不重组日志

```
memory/
├── logs/
│   └── 2026/
│       └── 03/
│           └── 2026-03-18.md    ← 今天的 append-only 日志
└── MEMORY.md                    ← 索引（夜间 distill 后才更新）
```

**日志格式示例：**

```markdown
## 2026-03-18 10:32

- [user] Senior data scientist, focuses on observability
- [feedback] User wants concise output without summaries
- [project] Merge freeze starts 2026-03-20

## 2026-03-18 14:15

- [feedback] Confirmed: real DB approach worked well, keep using it
- [project] Mobile release branch cut successfully
```

**为什么用 Daily Log？**

Assistant 模式是"永久会话"，用户可能连续对话几周甚至几个月。每次对话都重写 MEMORY.md 会导致冲突和丢失。用 append-only 日志，Agent 随时可以写，不需要协调；夜间 distill process 负责把日志整理成索引。

---

## 三、Team Memory：团队记忆共享

### 3.1 设计目标

多个团队成员共用同一份记忆。每个人都能读团队记忆，写自己的私有记忆。

### 3.2 目录结构

```
memory/                          ← 你的私有记忆
├── MEMORY.md
├── user/
└── ...

memory/team/                     ← 团队共享记忆
├── MEMORY.md
├── user/
├── feedback/
└── ...
```

### 3.3 记忆的 scope

```typescript
// src/memdir/memoryTypes.ts
// 有些记忆类型有 scope 标注
const TYPES_SECTION_COMBINED = [
  '<type>',
  '    <name>feedback</name>',
  '    <scope>default to private. team only when 明显是项目级公约</scope>',
  '</type>',
]
```

**判断标准**：
- 测试策略、构建规则 → team（每个人都该遵守）
- 个人风格偏好 → private（只对你生效）

---

## 四、记忆的 Distill 流程（夜间整理）

### 4.1 问题

Daily Log 模式下，日志越来越长。怎么从中提取出有用的索引？

### 4.2 Distill 流程

```
每夜（或按触发）
    ↓
读取过去 24h 的日志文件
    ↓
识别关键信息（用户偏好、项目决策、deadline）
    ↓
更新 MEMORY.md 索引
    ↓
可以删除已整理的日志（或归档）
```

具体实现因版本而异，可能是：
- 一个独立的 `dream` skill
- 一个定时 cron job
- 手动触发

### 4.3 为什么不用自动向量检索？

Distill 是**确定性**的整理（基于规则），不是语义搜索。好处是：
- 结果可预测、可审计
- 不依赖 embedding 质量
- 人也能读懂整理逻辑

---

## 五、记忆与 Session 的关系

### 5.1 单 Session 内的记忆

```
对话开始 → loadMemoryPrompt() → MEMORY.md 注入 system prompt
    ↓
Agent 运行时可以写新记忆
    ↓
对话结束 → 记忆文件已更新（但不是即时可见）
```

### 5.2 跨 Session 的记忆

下次对话开始时，`loadMemoryPrompt()` 读的是**磁盘上最新的 MEMORY.md**，所以上次对话写的新记忆对下次对话**立即可见**。

### 5.3 记忆的时效性

记忆可能过期（项目结束、人员变动）。cc-haha 的处理：

```typescript
// src/memdir/memoryTypes.ts
const MEMORY_DRIFT_CAVEAT = 
  'Memory records can become stale. Verify against current state before answering. 
   If a recalled memory conflicts with what you observe now, 
   trust what you observe now — and update or remove the stale memory.'
```

Agent 被要求**主动验证**记忆，而不是盲目相信它。

---

## 六、核心问题（学完应该能回答）

1. **记忆是什么时候加载进对话的？加载哪些内容？**
2. **Standard 模式和 Daily Log 模式的核心区别是什么？各自适合什么场景？**
3. **Daily Log 为什么要 append-only 而不是直接更新 MEMORY.md？**
4. **Team Memory 的目录结构是怎样的？private 和 team 记忆怎么共存？**
5. **如果一条记忆已经过时了，Agent 怎么处理？**

---

## 七、下一步

Day 4 我们学习**记忆搜索**：当 Agent 需要某个记忆时，是怎么找到的？Grep 的使用和 session transcript 的搜索。