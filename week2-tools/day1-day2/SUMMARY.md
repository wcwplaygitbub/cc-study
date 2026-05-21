# Week 2 Day 1-2 学习总结

> 学习日期：2026-05-18
> 学习时长：约 1.5 小时
> 理解程度：□ 完全理解  □ 大部分理解  □ 部分理解  □ 需要重学

---

## 一、当天问题记录

### Q1: 工具的三个核心字段是什么？各自分别起什么作用？

**A:**

| 字段 | 作用 |
|------|------|
| `name` | 工具唯一标识，LLM 决定调用哪个工具时用 |
| `description` | **LLM 主要靠它判断该不该用这个工具**，描述工具用途和用法 |
| `inputSchema` | 参数定义（JSON Schema），LLM 生成参数时参考，确保类型格式正确 |

### Q2: LLM 真的"理解"工具吗？它是怎么决定调用哪个工具的？

**A:** LLM 并不真正理解工具。它只是在匹配 description 中的语义模式。description 写了"Use for: file operations (ls, cat, mkdir)"，LLM 就在用户说"看看当前目录有什么文件"时想到用 BashTool。本质是**模式匹配 + 概率预测**，不是理解。

### Q3: BashTool 在执行命令前做了哪些安全检查？

**A:**

1. **安全检查（parseForSecurity）**：检查命令注入风险，如 `$(...)`, `` `...` `` 等危险语法
2. **权限检查（bashPermissions）**：根据 permission mode（allow/deny/bypass）决定是否允许执行
3. **路径验证（pathValidation）**：检查命令访问的路径是否在允许范围内
4. **沙箱检查（shouldUseSandbox）**：决定是否需要沙箱执行
5. **只读检查（readOnlyValidation）**：如果是只读模式，禁止修改操作

### Q4: buildTool 的作用是什么？它为工具定义了哪些默认行为？

**A:** `buildTool` 是工具构建工厂，为工具提供默认实现和标准化：
- 自动填充默认的 validate/execute/render 方法
- 把 inputSchema 转成 LLM 可读的 JSON Schema 格式
- 绑定 ToolUseContext 上下文

### Q5: 工具执行结果返回给 LLM 时，是什么角色？为什么？

**A:** `role: 'user'`。因为 tool_result 是外部注入的内容，不是 LLM 自身生成的 assistant 回复。用 user 角色最符合 API 语义，让 LLM 在下一轮把它当作上下文理解。

---

## 二、核心理解

### 工具定义三要素

name + description + inputSchema，LLM 靠 description 匹配场景，靠 inputSchema 生成参数。

### BashTool 执行链路

命令 → 安全解析 → 权限检查 → 路径验证 → 沙箱判断 → 执行 → 结果格式化 → 返回

### buildTool 工厂模式

统一工具创建入口，提供默认值填充、schema 转换、上下文绑定。

---

## 三、收获与疑问

### 今天最大的收获

理解了工具系统的本质不是"LLM 理解工具"，而是"LLM 通过 description 做模式匹配"。这解释了为什么工具 description 写得好不好直接决定了调用成功率。

### 还需要深入理解的地方

permission mode 的具体实现逻辑，以及 LocalShellTask 的子进程管理细节。

### 下一步学习计划

Day 3 深入 FileReadTool 和 FileEditTool，看文件操作类工具的实现。