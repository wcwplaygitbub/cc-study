# Week 2 Day 3 — 文件操作工具：FileReadTool 与 FileEditTool

> **核心问题**：Agent 是怎么读取和编辑文件的？diff 是怎么生成的？文件修改冲突怎么处理？

---

## 一、FileReadTool：文件读取

### 1.1 工具定义

```typescript
// src/tools/FileReadTool/FileReadTool.ts
export const FileReadTool = buildTool({
  name: FILE_READ_TOOL_NAME,
  
  inputSchema: z.object({
    file_path: z.string().describe('要读取的文件路径'),
    show_line_numbers: z.boolean().optional().describe('是否显示行号'),
    offset: z.number().optional().describe('从第几行开始读'),
    limit: z.number().optional().describe('限制读取行数'),
  }),
  
  async execute(params, ctx) {
    const { file_path, show_line_numbers, offset, limit } = params
    // 权限检查 → 路径解析 → 读取文件 → 返回结果
  }
})
```

### 1.2 权限检查

读取文件前要检查权限：

```typescript
// src/utils/permissions/filesystem.ts
export async function checkReadPermissionForTool(
  path: string,
  ctx: ToolUseContext
): Promise<PermissionDecision> {
  // 1. 检查路径是否在允许范围内（项目目录等）
  // 2. 检查 permission mode
  // 3. 返回 PermissionDecision
}
```

### 1.3 多种文件类型处理

FileReadTool 不只是读纯文本，还处理：

| 文件类型 | 处理方式 |
|---------|---------|
| **图片** | resize → base64 编码 → 作为 `image` 类型返回给 LLM |
| **PDF** | 提取文本/图片，使用 `extractPDFPages` |
| **笔记本（.ipynb）** | 解析成 cell 结构 |
| **普通文本** | 直接读取，支持 offset + limit 分段读取 |
| **二进制** | 返回 `[binary file]` 占位符 |

### 1.4 分段读取：大文件处理

```typescript
// src/utils/readFileInRange.ts
export async function readFileInRange(
  filePath: string,
  startLine: number,
  endLine: number
): Promise<{ content: string; totalLines: number }> {
  // 大文件不能一次性读入内存
  // 按行号范围读取，只返回需要的部分
  // 同时返回文件总行数，让 LLM 知道还有多少内容
}
```

### 1.5 Token 预算考虑

```typescript
// src/constants/apiLimits.ts
const PDF_AT_MENTION_INLINE_THRESHOLD = 3000  // 超过 3000 token 的 PDF 只保留引用
const PDF_MAX_PAGES_PER_READ = 10              // 最多一次读 10 页
```

读取大文件时会考虑 token 限制，避免超出 LLM 的上下文窗口。

---

## 二、FileEditTool：文件编辑

### 2.1 工具定义

```typescript
// src/tools/FileEditTool/FileEditTool.ts
export const FileEditTool = buildTool({
  name: FILE_EDIT_TOOL_NAME,
  
  inputSchema: z.object({
    file_path: z.string().describe('要编辑的文件绝对路径'),
    old_string: z.string().describe('要替换的原文'),
    new_string: z.string().describe('替换成什么'),
    replace_all: z.boolean().default(false).optional()
      .describe('是否替换所有匹配（默认 false）'),
  }),
  
  async execute(params, ctx) {
    // 权限检查 → 读取原文件 → 生成 patch → 写入 → 返回 diff
  }
})
```

### 2.2 编辑执行流程

```typescript
async execute(params, ctx) {
  const { file_path, old_string, new_string, replace_all } = params
  
  // 1. 权限检查
  const perm = await checkWritePermissionForTool(file_path, ctx)
  if (!perm.allowed) {
    return { error: perm.message }
  }
  
  // 2. 读取原文件
  const originalContent = await readFileSync(file_path, 'utf-8')
  
  // 3. 检查文件是否被外部修改（staleness check）
  const fileMtime = getFileModificationTime(file_path)
  if (params.fileMtime && fileMtime !== params.fileMtime) {
    return { error: FILE_UNEXPECTIALLY_MODIFIED_ERROR }
  }
  
  // 4. 执行替换
  const editedContent = replace_all
    ? originalContent.replaceAll(old_string, new_string)
    : originalContent.replace(old_string, new_string)
  
  // 5. 写入文件
  await writeTextContent(file_path, editedContent)
  
  // 6. 生成 git diff（如果有 git）
  const diff = await fetchSingleFileGitDiff(file_path)
  
  // 7. 返回结果
  return {
    filePath: file_path,
    originalFile: originalContent,
    newFile: editedContent,
    gitDiff: diff,
  }
}
```

### 2.3 核心：old_string 的精确匹配

old_string 必须**精确匹配**文件中的原文才能替换。这和 sed 不同：

```bash
# sed 可以模糊匹配
sed -i 's/old_string/new_string/g' file

# FileEditTool 必须 exact match
old_string = "  name: string"      # ✓ 精确匹配
old_string = "name: string"       # ✗ 空格不对，匹配失败
```

### 2.4 replace_all vs 单次替换

```typescript
// 单次替换（默认）
originalContent.replace(old_string, new_string)

// 全局替换
originalContent.replaceAll(old_string, new_string)
```

### 2.5 文件并发修改检测（staleness check）

```typescript
// types.ts 中
const FILE_UNEXPECTEDLY_MODIFIED_ERROR = 
  'File was modified after the read. Please re-read and try again.'

// execute 中
if (params.fileMtime && fileMtime !== params.fileMtime) {
  // LLM 读取文件之后、编辑之前，有人在外面改了文件
  // 拒绝执行，返回错误，让 LLM 重新读取再尝试
  return { error: FILE_UNEXPECTEDLY_MODIFIED_ERROR }
}
```

### 2.6 Diff 生成

```typescript
// src/utils/diff.ts
export function countLinesChanged(
  original: string, 
  edited: string
): { additions: number; deletions: number } {
  // 对比两个字符串，生成行级别的 diff
}

// src/utils/gitDiff.ts
export async function fetchSingleFileGitDiff(
  filePath: string
): Promise<ToolUseDiff | null> {
  // 如果文件在 git 仓库里，生成标准 unified diff 格式
}
```

---

## 三、文件工具的权限模型

### 3.1 权限检查入口

```typescript
// src/utils/permissions/filesystem.ts
export async function matchingRuleForInput(
  path: string,
  tool: string,
  operation: 'read' | 'write'
): Promise<PermissionRule | null>

export async function checkReadPermissionForTool(
  path: string,
  ctx: ToolUseContext
): Promise<PermissionDecision>

export async function checkWritePermissionForTool(
  path: string,
  ctx: ToolUseContext
): Promise<PermissionDecision>
```

### 3.2 权限决策结果

```typescript
type PermissionDecision = {
  allowed: boolean
  reason?: string       // denied 时的原因
  rule?: PermissionRule // 匹配的规则
}
```

### 3.3 通配符匹配

```typescript
// src/utils/permissions/shellRuleMatching.ts
export function matchWildcardPattern(
  pattern: string,  // e.g. "src/**/*.ts"
  path: string
): boolean {
  // 支持 **, *, ? 等通配符
  // 判断 path 是否匹配 pattern
}
```

---

## 四、文件编辑 vs Bash sed

| | FileEditTool | Bash sed |
|---|---|---|
| **匹配方式** | exact string match | regex |
| **替换策略** | 单次或全局 | 全局 |
| **冲突检测** | ✅ mtime 检查 | ❌ 无 |
| **权限检查** | 完整权限模型 | 仅靠 shell 本身 |
| **原子性** | 高（先读后写） | 低（sed 直接改） |
| **可逆性** | 记录 original | 无记录 |

---

## 五、Git Diff 输出示例

FileEditTool 的返回包含完整的 git diff：

```typescript
// output type
interface FileEditOutput {
  filePath: string
  originalFile: string
  newString: string
  oldString: string
  structuredPatch: {
    oldStart: number
    oldLines: number
    newStart: number
    newLines: number
    lines: string[]
  }[]
  gitDiff?: {
    filename: string
    status: 'modified' | 'added'
    additions: number
    deletions: number
    patch: string  // unified diff 格式
  }
}
```

---

## 六、核心问题（学完应该能回答）

1. FileReadTool 读取大文件时怎么避免超出 token 限制？
2. FileEditTool 的 old_string 匹配和 sed 有什么区别？
3. staleness check 是怎么防止并发修改冲突的？
4. FileEditTool 返回的 structuredPatch 是什么结构？
5. 文件工具的权限检查和 BashTool 权限检查有什么区别？

---

## 七、下一步

Day 4 我们学习 WebSearchTool 和 WebFetchTool，了解网络相关工具的实现。