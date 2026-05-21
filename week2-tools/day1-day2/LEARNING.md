# Week 2 Day 1-2 — 工具系统基础：Tool 接口与 BashTool

> **核心问题**：工具是怎么定义的？LLM 是怎么"知道"有哪些工具以及怎么调用的？BashTool 是怎么执行命令并返回结果的？

---

## 一、为什么需要工具系统

Agent Loop 只能推理和回复文字。要让它真正"做事"（读文件、执行命令、搜索网页），就需要**工具系统**。

工具系统的核心职责：
1. 告诉 LLM 有哪些工具可用（工具注册）
2. 把 LLM 的调用请求转换成实际执行（工具分发）
3. 把执行结果返回给 LLM（结果处理）

---

## 二、Tool 接口：工具的定义

### 2.1 三个核心字段

cc-haha 中所有工具都实现统一的 `Tool` 接口，最关键的三个字段：

```typescript
// src/Tool.ts
interface Tool {
  // 工具唯一标识，LLM 通过这个名字决定调用哪个
  readonly name: string
  
  // 工具用途描述，LLM 靠它理解什么场景用这个工具
  description(ctx: ToolUseContext): string
  
  // 参数 schema（JSON Schema 格式），LLM 靠它知道怎么传参
  readonly inputSchema: Input
}
```

### 2.2 一个完整例子：BashTool

```typescript
// src/tools/BashTool/BashTool.tsx
export const BASH_TOOL_NAME = 'Bash'

export const build = buildTool({
  name: BASH_TOOL_NAME,
  
  description: (ctx) => 
    `Execute bash/shell commands. Use for: file operations (ls, cat, mkdir),
     git operations, running scripts, package management, searching files (grep),
     viewing outputs, and any system operations. Returns stdout + stderr.`,
  
  inputSchema: z.object({
    command: z.string().describe('The shell command to execute'),
    timeout: z.number().optional().describe('Timeout in milliseconds'),
  }),
  
  // 实际执行逻辑
  async execute(params, ctx) {
    const { command, timeout = 30000 } = params
    // 权限检查
    // 路径验证
    // 沙箱检查
    // 执行命令
    // 返回结果
  }
})
```

### 2.3 工具定义的三个字段 vs LLM 的关系

| 字段 | 作用 | LLM 行为 |
|------|------|---------|
| `name` | 唯一标识 | LLM 决定调用哪个工具时用 |
| `description` | 工具用途+用法 | **LLM 主要靠它判断该不该用这个工具** |
| `inputSchema` | 参数定义 | LLM 生成参数时参考，确保类型格式正确 |

**LLM 并不"理解"工具**，它只是在匹配 description 中的语义模式。比如 description 里写了"Use for: file operations (ls, cat, mkdir)"，LLM 就会在用户说"看看当前目录有什么文件"时想到用这个工具。

---

## 三、buildTool 工厂函数

`buildTool` 是 cc-haha 提供的工具构建器，提供默认值和标准化：

```typescript
// src/Tool.ts
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D>
```

它做的事情：
1. 自动填充默认实现（如果没有提供 `validate`/`execute` 等方法）
2. 把 inputSchema 转成 LLM 可读的格式
3. 绑定上下文（ToolUseContext）

```typescript
// buildTool 支持的工具方法
interface ToolDef {
  name: string
  aliases?: string[]           // 别名，向后兼容
  description: string | ((ctx: ToolUseContext) => string)
  inputSchema: z.ZodType | ToolInputJSONSchema
  
  // 可选方法
  validate?: (params: unknown, ctx: ToolUseContext) => ValidationResult
  execute?: (params: unknown, ctx: ToolUseContext) => Promise<ToolResult>
  render?: (progress: ToolProgress) => JSX.Element  // 进度展示
  authorize?: (ctx: ToolUseContext) => Promise<AuthorizationResult>
}
```

---

## 四、BashTool 执行流程详解

### 4.1 入口：execute 方法

```typescript
// BashTool.tsx 的 execute 核心逻辑
async execute(params, ctx) {
  // 1. 解析参数
  const { command, timeout = 30000 } = params
  
  // 2. 安全检查
  const securityResult = parseForSecurity(command)
  if (!securityResult.safe) {
    return { error: securityResult.reason }
  }
  
  // 3. 权限检查
  const permission = await checkPermission(command)
  if (permission.denied) {
    return { error: permission.message }
  }
  
  // 4. 路径验证
  const pathValidation = validatePaths(command)
  
  // 5. 执行命令
  const result = await spawnShellTask(command, { timeout })
  
  // 6. 返回结果
  return formatResult(result)
}
```

### 4.2 安全检查：parseForSecurity

防止命令注入，检查：
- 危险的 shell 语法（`$(...)`, `` `...` `` 等命令替换）
- 多余的 sudo/管理员权限请求
- 可疑的多余空格/字符

### 4.3 权限检查：bashPermissions

```typescript
// src/tools/BashTool/bashPermissions.ts
export async function bashToolHasPermission(
  command: string,
  ctx: ToolUseContext
): Promise<PermissionResult> {
  // 读取当前 permission mode（allow/deny/bypass）
  // 根据命令类型和 permission mode 决定是否允许
}
```

cc-haha 支持多种权限模式：
- `allow` — 全部允许
- `deny` — 全部拒绝（除非明确批准）
- `bypass` — 跳过权限检查（开发者模式）

### 4.4 沙箱检查：shouldUseSandbox

检查是否应该用沙箱执行命令：

```typescript
// src/tools/BashTool/shouldUseSandbox.ts
export function shouldUseSandbox(command: string): boolean {
  // 危险命令（rm -rf /）→ 强制沙箱
  // 读写项目文件 → 可能需要沙箱
  // 普通查询命令 → 不需要
}
```

### 4.5 命令执行：spawnShellTask

```typescript
// src/tasks/LocalShellTask/LocalShellTask.ts
export function spawnShellTask(
  command: string,
  options: { timeout?: number; cwd?: string }
): Task {
  // 创建子进程执行命令
  // 支持 stdout/stderr 捕获
  // 支持超时控制
  // 返回 Task 对象，可监控进度
}
```

### 4.6 结果展示：UI 渲染

BashTool 的 execute 返回后，结果会通过 React 组件渲染给用户：

```typescript
// src/tools/BashTool/UI.tsx
export function renderToolResultMessage(result: BashResult): JSX.Element {
  // 根据命令类型决定怎么展示
  // grep/cat 等读命令 → 折叠显示
  // ls 命令 → 目录列表格式
  // 危险命令 → 高亮警告
}
```

---

## 五、工具是怎么注册到 Agent 的

工具注册不是 cc-haha 直接做的，而是通过 `tools.ts` 统一暴露：

```typescript
// src/tools.ts
export function getTools(state: AppState): Tool[] {
  return [
    BashTool,
    FileReadTool,
    FileEditTool,
    WebSearchTool,
    // ... 更多工具
  ]
}
```

这些工具在 Agent Loop 初始化时传入：

```typescript
// src/query.ts
const tools = getTools(state)
const toolDefs = tools.map(tool => tool.toToolDef())

// 发给 LLM 的请求
const response = await callLLM({
  model,
  messages,
  tools: toolDefs  // ← 工具定义列表
})
```

---

## 六、工具执行结果的返回格式

cc-haha 定义了统一的 `ToolResultBlockParam`：

```typescript
// @anthropic-ai/sdk/resources/index.mjs
interface ToolResultBlockParam {
  type: 'tool_result'
  content: string | ({
    type: 'image'
    source: { type: 'base64'; media_type: string; data: string }
  } | {
    type: 'text'
    text: string
  })[]
  tool_use_id: string  // 对应 LLM 原始 tool_use 块的 id
}
```

---

## 七、核心问题（学完应该能回答）

### Q1: 工具的三个核心字段是什么？各自分别起什么作用？

### Q2: LLM 真的"理解"工具吗？它是怎么决定调用哪个工具的？

### Q3: BashTool 在执行命令前做了哪些安全检查？

### Q4: buildTool 的作用是什么？它为工具定义了哪些默认行为？

### Q5: 工具执行结果返回给 LLM 时，是什么角色？为什么？

---

## 八、下一步

Day 3 我们深入 `FileReadTool` 和 `FileEditTool`，看看文件操作类工具是怎么实现的。