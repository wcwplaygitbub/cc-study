# Week 3 Day 4 — 记忆搜索与 Session 管理

> **核心问题**：Agent 怎么找到特定的记忆？Session 之间怎么区分？transcript 日志是什么？

---

## 一、记忆搜索机制

### 1.1 索引 + Grep = 记忆搜索

cc-haha 没有向量检索，只用**索引 + Grep**：

```
MEMORY.md（索引）
  ↓
grep 关键字找到对应记忆文件
  ↓
读取文件内容
  ↓
作为上下文给 LLM
```

### 1.2 Grep 工具搜索记忆

```typescript
// src/memdir/memdir.ts
export function buildSearchingPastContextSection(memoryDir: string): string[] {
  return [
    '## Searching past context',
    '',
    'When looking for past context:',
    '1. Search topic files in your memory directory:',
    `   grep -rn "<search term>" ${memoryDir} --include="*.md"`,
    '2. Session transcript logs (last resort — large files, slow):',
    `   grep -rn "<search term>" ${projectDir}/ --include="*.jsonl"`,
    '',
    'Use narrow search terms (error messages, file paths, function names).',
  ]
}
```

**为什么不用向量搜索？**
- cc-haha 的记忆是**文件化**的，grep 已经够用
- 向量检索依赖 embedding 模型，额外依赖
- 文件可以直接用文本工具搜索，不需要额外基础设施

### 1.3 搜索的优先级

```
记忆目录（memory/*.md） ← 优先搜这里，规模小、相关度高
    ↓ 没找到
Session Transcript（*.jsonl）← 最后手段，文件大、搜索慢
```

---

## 二、Session 管理

### 2.1 Session 的定义

cc-haha 中一个 Session = 一次 `cc-haha` 启动到关闭的过程。每次启动是新的 Session，每次 `/new` 也算新 Session。

```typescript
// src/types/ids.ts
export type SessionId = string  // 格式: session_xxx 或类似
```

### 2.2 Session 的目录结构

```
~/.claude/projects/<slug>/
├── memory/              ← 持久化记忆（跨 Session）
│   ├── MEMORY.md
│   └── [type]/          ← 各类记忆文件
├── sessions/            ← 各 Session 的 transcript
│   ├── session_abc123/
│   │   └── transcript.jsonl
│   └── session_def456/
│       └── transcript.jsonl
└── state/               ← Session 内的临时状态
```

### 2.3 Transcript 日志

每次 Session 的对话内容会记录到 `.jsonl` 文件：

```
{"role": "user", "content": "...", "timestamp": 1710000000}
{"role": "assistant", "content": "...", "timestamp": 1710000001}
{"role": "tool_use", "tool": "BashTool", "input": {...}}
{"role": "tool_result", "content": "..."}
```

Transcript 的用途：
1. **Session 内回顾**：Agent 可以 `grep` 自己的历史消息
2. **审计**：人可以直接读 jsonl 看 Agent 做了什么
3. **上下文恢复**：如果 Session 断开，可以从 transcript 恢复

### 2.4 Session 隔离

| | Session 内 | Session 间 |
|---|---|---|
| **对话历史** | 共享（在 transcript 里） | 不共享（除非显式加载） |
| **记忆文件** | 共享 | 共享 |
| **临时状态** | 独立 | 不传递 |

---

## 三、Session 与记忆的交互

### 3.1 当前 Session 写入新记忆

```
Agent 决定保存记忆
    ↓
写文件到 memory/[type]/
    ↓
更新 memory/MEMORY.md
    ↓
记忆立即持久化
```

### 3.2 后续 Session 读取该记忆

```
新 Session 开始
    ↓
loadMemoryPrompt() 读取 MEMORY.md
    ↓
Agent 看到新记忆的索引
    ↓
如果需要详情，grep 找到文件并读取
```

**关键**：记忆的写入和读取是**文件系统级别**的持久化，不依赖 Session。

---

## 四、Session 的创建与切换

### 4.1 新建 Session

```typescript
// cc-haha 启动时
const sessionId = generateSessionId()
// 初始化 session 目录和 state
// 加载 MEMORY.md
```

### 4.2 切换 Session

```
用户输入 /new 或 /session switch
    ↓
保存当前 Session 状态
    ↓
创建新 Session 或切换到已有 Session
    ↓
加载新 Session 的 MEMORY.md
```

### 4.3 Session 列表

cc-haha 可以查看所有历史 Session：

```
sessions/
├── session_20260115_abc/
├── session_20260220_def/
└── session_20260318_ghi/
```

---

## 五、记忆失效与清理

### 5.1 过时记忆的检测

```typescript
// src/memdir/memoryTypes.ts
export const MEMORY_DRIFT_CAVEAT = [
  'Memory records can become stale over time.',
  'Before answering based solely on a memory, verify it is still correct.',
  'If a recalled memory conflicts with current info, trust what you observe now.',
]
```

Agent 被要求**在使用记忆前验证**，而不是盲目相信。

### 5.2 主动更新/删除

```typescript
// 如果用户说"ignore that memory"
Agent 应该在 MEMORY.md 中：
1. 删除/注释掉对应的索引行
2. 或者更新记忆文件
```

### 5.3 归档旧 Session

大量 Session 的 transcript 会占用空间。cc-haha 没有自动清理，用户需要手动管理。

---

## 六、核心问题（学完应该能回答）

1. **cc-haha 用什么方式搜索记忆？为什么不用向量检索？**
2. **Session 的 transcript 是什么格式？有什么用？**
3. **Session 之间共享什么、不共享什么？**
4. **如果 Agent 读了一条过时的记忆，它应该怎么处理？**
5. **Session 的目录结构和记忆的目录结构是什么关系？**

---

## 七、下一步

Day 5 我们来聊聊**记忆系统的局限性和设计权衡**，以及 cc-haha 和 OpenHuman 记忆系统的对比。