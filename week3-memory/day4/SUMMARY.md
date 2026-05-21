# Week 3 Day 4 学习总结

> 学习日期：2026-05-18
> 学习时长：约 1 小时
> 理解程度：□ 完全理解  □ 大部分理解  □ 部分理解  □ 需要重学

---

## 一、当天问题记录

### Q1: cc-haha 用什么方式搜索记忆？为什么不用向量检索？

**A:** 用 Grep 搜索记忆目录，优先搜 memory/*.md，找不到才搜 transcript 日志（*.jsonl）。不用向量检索是因为记忆文件小且文本化，grep + 索引已经够用；避免 embedding 依赖；结果可预测可审计。

### Q2: Session 的 transcript 是什么格式？有什么用？

**A:** JSONL 格式，每行一条消息（role/content/timestamp）。用途：1）Session 内回顾历史；2）审计；3）Session 断开时上下文恢复。

### Q3: Session 之间共享什么、不共享什么？

**A:** 共享：记忆文件（memory/*.md）。不共享：对话历史 transcript、临时状态。每次新 Session 只加载 MEMORY.md，不加载历史 transcript。

### Q4: Agent 读了一条过时记忆怎么处理？

**A:** 信任当前观察到的状态，而不是记忆。更新或删除过时记忆。

### Q5: Session 的目录结构和记忆的目录结构是什么关系？

**A:** 平级关系，都在 `~/.claude/projects/<slug>/` 下：
- `memory/` — 跨 Session 持久化
- `sessions/<session_id>/` — 单 Session 的 transcript 和状态

---

## 二、签字确认

Week 3 完成：□ 是
继续 Week 4（Skills 系统）：□ 是