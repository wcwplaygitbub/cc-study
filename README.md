# 构建 Claude Code 级 Agent — 学习计划

> 通过 cc-haha 项目源码，系统性学习一个工业级 AI Agent 是怎么设计的

## 背景

cc-haha 是基于 Claude Code 泄露源码重构的项目，保留了完整的源码结构和注释，是研究 Agent 工程实现的绝佳样本。

## 目标

理解一个工业级 Agent（类 Claude Code）的核心模块设计，为自己开发类似系统积累经验。

**不是**：学会用 Claude Code
**是**：学会设计和实现 Claude Code 级别的 Agent 系统

## 学习方式

1. 每天我准备学习资料（逻辑讲解、流程图、伪代码）
2. 你阅读资料，有问题随时问我
3. 每天结束后把总结沉淀到当天的文件夹
4. 第二天继续下一个模块

## 课程表（6周）

| 周 | 模块 | 核心问题 | 时间 |
|----|------|---------|------|
| **Week 1** | Agent Loop | Agent是怎么跑起来的？消息→思考→行动→循环的完整流程 | 7天 |
| **Week 2** | 工具系统 | Agent怎么用工具与外界交互？Bash/Edit/Grep怎么实现的？ | 7天 |
| **Week 3** | 记忆系统 | Agent怎么跨会话记住重要信息？记忆怎么注入到上下文中？ | 7天 |
| **Week 4** | 扩展系统 | Skills和斜杠命令怎么让Agent能力可扩展？ | 7天 |
| **Week 5** | 多Agent系统 | 一个主Agent怎么协调多个子Agent并行工作？ | 7天 |
| **Week 6** | 集成与工程化 | 怎么把各模块组合成完整系统？IM接入/桌面端/质量门禁 | 7天 |

## 每周目标

### Week 1 — Agent Loop（核心引擎）⭐ 最核心的一周
理解 Agent 的"发动机"是怎么工作的

### Week 2 — 工具系统
理解 Agent 的"手和脚"——通过工具操作文件、执行命令

### Week 3 — 记忆系统
理解 Agent 的"海马体"——跨会话持久化

### Week 4 — Skills系统
理解 Agent 的"技能卡"——可插拔的能力扩展

### Week 5 — 多Agent
理解 Agent 的"团队协作"——主从协调模式

### Week 6 — 集成工程化
理解怎么把各模块组合成完整产品

## 文件夹结构

```
cc-haha-learning/
├── README.md                    # 本文件
├── week1-agent-loop/
│   ├── day1-day2/              # Agent Loop 主循环
│   ├── day3-day5/              # 消息构建/Token管理/工具执行
│   └── day6-day7/              # 实战+复盘
├── week2-tools/
├── week3-memory/
├── week4-skills/
├── week5-multi-agent/
└── week6-integration/
```

## 如何开始

从 **Week 1 Day 1** 开始，先读当天文件夹里的 `LEARNING.md`，然后开始学习。
