# Week 3 Day 1-2 — 记忆系统基础：MEMORY.md 与记忆类型

> **核心问题**：Agent 的记忆是怎么持久化的？MEMORY.md 是什么？为什么需要分类记忆？哪些东西不该记？

---

## 一、为什么需要记忆系统

没有记忆的 Agent，每次对话都是从零开始——不知道用户是谁、不知道之前做过什么、不知道有什么偏好。

记忆系统的目标：**让 Agent 能跨 Session 记住重要信息，下一次对话时自动继承上下文。**

cc-haha 的记忆是**文件化的**：不是存在数据库里，而是直接写在 `*.md` 文件里，Agent 通过读这些文件来"想起"之前的事。

---

## 二、MEMORY.md：记忆的入口文件

### 2.1 核心设计

```
memory/
├── MEMORY.md      ← 入口索引（每次对话自动加载）
├── user/          ← 用户记忆
├── feedback/      ← 反馈记忆
├── project/       ← 项目记忆
└── reference/     ← 外部系统引用记忆
```

**MEMORY.md 是索引，不是存储**。每个条目是一行引用：

```markdown
- [user_role](user/user_role.md) — Senior engineer, prefers detailed technical explanations
- [feedback_bash_timeout](feedback/feedback_bash_timeout.md) — Increase timeout for long builds
```

### 2.2 加载时机

每次对话开始时，Agent 会自动把 MEMORY.md 读进来，作为 system prompt 的一部分：

```typescript
// src/memdir/memdir.ts
export async function loadMemoryPrompt(): Promise<string | null> {
  const autoEnabled = isAutoMemoryEnabled()
  if (!autoEnabled) return null
  
  // 读取 MEMORY.md 内容，构建到 system prompt
  const entrypoint = memoryDir + 'MEMORY.md'
  let entrypointContent = fs.readFileSync(entrypoint, 'utf-8')
  
  // 如果太长就截断（最多 200 行，25KB）
  const t = truncateEntrypointContent(entrypointContent)
  return buildMemoryPrompt({ entrypoint: t.content, ... })
}
```

### 2.3 为什么用文件而不是数据库？

| | 文件（cc-haha） | 数据库（OpenHuman） |
|---|---|---|
| 可见性 | 人可以直接打开看/改 | 不可见，需工具查询 |
| 可编辑 | Agent 能写，人也能写 | 只能通过工具读写 |
| 持久化 | 跟着项目走，git 可追踪 | 需要另外备份 |
| 检索 | 简单 grep 就行 | 需要向量检索 |
| 适用场景 | 个人/小团队记忆 | 大规模知识库 |

cc-haha 的方式是"**记忆即文档**"，人、Agent、Git 都能理解和操作同一份文件。

---

## 三、记忆分类（Memory Taxonomy）

### 3.1 四种记忆类型

cc-haha 的记忆被严格分成 4 类，每类有明确的用途和保存时机：

| 类型 | 英文名 | 什么时候记 | 什么时候用 |
|------|--------|-----------|-----------|
| **用户** | user | 用户告诉你他的角色、目标、知识背景 | 理解用户是谁，调整回答方式 |
| **反馈** | feedback | 用户纠正你/确认你的做法 | 避免重复犯错，记住有效做法 |
| **项目** | project | 项目进展、deadline、决策 | 理解上下文，预测需求 |
| **引用** | reference | 外部系统（Linear、Slack、Grafana）| 指向外部查找最新信息 |

### 3.2 用户记忆（user）

**示例：**
```markdown
---
name: senior_data_scientist
description: Senior data scientist focused on observability/logging
type: user
---

User is a data scientist. When explaining code, frame in terms of 
data pipelines and metrics rather than generic web dev concepts.

User prefers verbose output with actual numbers, not summary statements.
```

**保存时机**：用户透露任何关于自己角色、偏好、知识背景的信息。

### 3.3 反馈记忆（feedback）

**结构**：规则 → Why → How to apply

```markdown
---
name: no_database_mocks
description: Integration tests must use real database, not mocks
type: feedback
---

Integration tests must hit a real database, not mocks.

**Why:** Past incident where mocked tests passed but the prod migration
failed because mock/prod diverged silently.

**How to apply:** When writing tests for this project, always use
real database. Reject or question any test that uses mock DB setup.
```

**保存时机**：用户纠正你（"不是这样"）或确认你的做法（"对，这样很好"）。

### 3.4 项目记忆（project）

**结构**：事实 → Why → How to apply

```markdown
---
name: merge_freeze_20260305
description: Merge freeze for mobile release branch
type: project
---

Merge freeze begins 2026-03-05 for mobile release cut.

**Why:** Mobile team needs stable branch to cut release.

**How to apply:** Flag any non-critical PR work scheduled after that
date. Do not suggest large refactors in this period.
```

**保存时机**：你了解到项目的进展、deadline、重大决策。

### 3.5 引用记忆（reference）

```markdown
---
name: linear_ingest_project
description: Pipeline bugs tracked in Linear project "INGEST"
type: reference
---

Pipeline bugs are tracked in Linear project "INGEST".
URL: https://linear.app/org/project/INGEST

Check here for context on pipeline-related tickets.
```

**保存时机**：你发现某个信息存在外部系统时，记住"去哪儿找"而不是"内容本身"。

---

## 四、哪些东西不该记

```typescript
// src/memdir/memoryTypes.ts
const WHAT_NOT_TO_SAVE_SECTION = [
  '## What NOT to save in memory',
  '',
  '- Code patterns, conventions, architecture — 可从代码推导',
  '- Git history — git log / git blame 更权威',
  '- Debug 方案 — 修复在代码里，commit message 有上下文',
  '- 已写在 CLAUDE.md 里的东西',
  '- 临时任务细节、进行中的工作、当前对话上下文',
]
```

**核心原则**：记忆只存"不可从当前项目状态推导"的信息。

---

## 五、记忆的读写流程

### 5.1 读流程（每次对话）

```
对话开始
  ↓
loadMemoryPrompt() 读取 MEMORY.md
  ↓
构建 system prompt，注入记忆行
  ↓
Agent 看到索引，知道有哪些记忆文件
  ↓
如果需要某条记忆，用工具读对应文件
```

### 5.2 写流程（发现新信息时）

```
发现值得记住的信息
  ↓
判断类型（user/feedback/project/reference）
  ↓
写新文件（如 feedback/my_feedback.md）
  ↓
更新 MEMORY.md（加一行索引）
```

**写两步骤**：先写单独的记忆文件，再更新索引。MEMORY.md 永远只是索引，不能直接往里写内容。

---

## 六、核心问题（学完应该能回答）

1. **MEMORY.md 和普通 .md 记忆文件有什么区别？为什么这样设计？**
2. **cc-haha 把记忆分成哪 4 类？各自的用途是什么？**
3. **为什么"代码模式、Git 历史、项目结构"这类信息不应该存记忆？**
4. **feedback 记忆的结构是什么？为什么要有 Why 和 How to apply？**
5. **记忆的写流程是什么？为什么分成两步？**

---

## 七、下一步

Day 3 我们深入了解**记忆的加载时机**、**AutoMemory 的 daily log 模式**，以及**Session 之间的记忆共享机制**。