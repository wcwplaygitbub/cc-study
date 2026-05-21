# Week 3 Day 3 学习总结

> 学习日期：2026-05-18
> 学习时长：约 1 小时
> 理解程度：□ 完全理解  □ 大部分理解  □ 部分理解  □ 需要重学

---

## 一、当天问题记录

### Q1: 记忆是什么时候加载进对话的？加载哪些内容？

**A:** 对话开始时，`loadMemoryPrompt()` 在构建 system prompt 阶段被调用，读取 MEMORY.md 内容，注入 system prompt。加载内容包括：记忆行为指导 + MEMORY.md 索引（被截断到 200行/25KB 以内）。

### Q2: Standard 模式和 Daily Log 模式的核心区别是什么？

**A:** Standard 直接写文件和更新 MEMORY.md；Daily Log 是 append-only 日志，只追加不重写，MEMORY.md 由夜间 distill 流程更新。Daily Log 适合"永久会话"场景，避免并发写入冲突。

### Q3: Daily Log 为什么要 append-only？

**A:** 永久会话中多轮对话可能跨午夜，直接更新 MEMORY.md 会导致频繁读写冲突和丢失。Append-only 日志没有冲突问题，夜间统一整理。

### Q4: Team Memory 的目录结构是怎样的？

**A:** `memory/` 是私有记忆，`memory/team/` 是团队共享记忆。private 和 team 记忆通过目录分开，MEMORY.md 可以有 combined 和 individual 两种格式。

### Q5: 如果记忆过时了，Agent 怎么处理？

**A:** Agent 被要求在使用记忆前**主动验证**（看当前文件/资源是否还和记忆一致）。如果冲突，信任观察到的现状，并更新或删除过时记忆。

---

## 二、核心理解

### 截断机制

200 行 + 25KB 上限保证了每次加载的可控成本，不会因为记忆太多导致 token 超限。

### Daily Log 的适用场景

不是给普通代码工具用的，是给"永久在线助手"设计的，避免了多天连续会话中的记忆管理复杂性。

---

## 三、签字确认

Week 3 完成：□ 是
继续 Week 4（Skills 系统）：□ 是