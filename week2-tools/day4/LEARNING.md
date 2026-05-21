# Week 2 Day 4 — 网络工具：WebSearchTool 与 WebFetchTool

> **核心问题**：Agent 是怎么搜索网页的？WebFetchTool 是怎么抓取并解析网页内容的？为什么需要两个工具配合使用？

---

## 一、为什么需要两个工具

| 工具 | 作用 | 输出 |
|------|------|------|
| WebSearchTool | 搜索关键字，返回相关网页链接列表 | `[ {title, url}, ... ]` |
| WebFetchTool | 抓取指定 URL，提取可读内容 | 页面正文 + prompt 处理结果 |

**工作流程**：WebSearchTool 找到相关网页 → WebFetchTool 抓取具体页面内容 → LLM 阅读理解后回答用户。

---

## 二、WebSearchTool：网页搜索

### 2.1 工具定义

```typescript
// src/tools/WebSearchTool/WebSearchTool.ts
const inputSchema = z.strictObject({
  query: z.string().min(2).describe('The search query to use'),
  allowed_domains: z.array(z.string()).optional()
    .describe('Only include search results from these domains'),
  blocked_domains: z.array(z.string()).optional()
    .describe('Never include search results from these domains'),
})

export const WebSearchTool = buildTool({
  name: WEB_SEARCH_TOOL_NAME,
  
  inputSchema,
  
  async execute(params, ctx) {
    const { query, allowed_domains, blocked_domains } = params
    // 调用搜索 API → 格式化结果 → 返回
  }
})
```

### 2.2 多后端支持

WebSearchTool 支持多个搜索提供商：

```typescript
// src/tools/WebSearchTool/backend.ts
export async function resolveWebSearchProvider() {
  // 1. Anthropic 原生搜索（如果有的话）
  // 2. Fallback 到第三方 API（搜索 API）
  // 3. 返回可用的 provider
}

export async function searchWithExternalProvider(
  query: string,
  options: { allowed_domains?, blocked_domains? }
) {
  // 调用外部搜索 API
  // 返回格式化的搜索结果
}
```

### 2.3 搜索结果格式

```typescript
// output schema
const outputSchema = z.object({
  query: string,                              // 实际执行的搜索词
  results: z.array(z.union([
    { title: string, url: string },           // 搜索命中
    z.string()                                // 模型生成的文字说明
  ])),
  durationSeconds: number,                     // 耗时
})
```

### 2.4 native Web Search vs 第三方 API

cc-haha 优先用 **Anthropic native web search**（如果有的话），没有的话 fallback 到第三方搜索 API。

```typescript
// backend.ts
export function isWebSearchEnabledForModel(model: string): boolean {
  // 检查当前模型是否支持原生 web search
}

export function shouldFallbackFromNativeError(error): boolean {
  // 原生搜索出错时，是否 fallback 到第三方
}
```

---

## 三、WebFetchTool：网页抓取

### 3.1 工具定义

```typescript
// src/tools/WebFetchTool/WebFetchTool.ts
const inputSchema = z.strictObject({
  url: z.string().url().describe('The URL to fetch content from'),
  prompt: z.string().describe('The prompt to run on the fetched content'),
})

export const WebFetchTool = buildTool({
  name: WEB_FETCH_TOOL_NAME,
  
  inputSchema,
  
  async execute(params, ctx) {
    const { url, prompt } = params
    // 权限检查 → 获取页面 → 处理内容 → 执行 prompt → 返回
  }
})
```

### 3.2 权限检查：preapproved hosts

```typescript
// src/tools/WebFetchTool/preapproved.ts
export function isPreapprovedHost(hostname: string): boolean {
  // 白名单：公共可访问的网站不需要额外确认
  const preapproved = [
    'github.com',
    'stackoverflow.com',
    'wikipedia.org',
    'docs.anthropic.com',
    // ... 更多
  ]
  return preapproved.includes(hostname)
}
```

### 3.3 内容获取流程

```typescript
// src/tools/WebFetchTool/utils.ts
export async function getURLMarkdownContent(url: string): Promise<FetchedContent> {
  // 1. 发送 HTTP GET 请求
  // 2. 检测内容类型（HTML → 解析提取正文；其他 → 直接返回）
  // 3. HTML 解析：
  //    - 提取 <article>, <main> 等语义标签
  //    - 移除 script, style, nav, footer 等噪音
  //    - 提取纯文本内容
  // 4. 限制最大长度（MAX_MARKDOWN_LENGTH = 100_000）
  // 5. 返回 markdown 格式内容
}
```

### 3.4 prompt 处理

fetch 到的原始内容不直接返回，而是经过 prompt 处理：

```typescript
// src/tools/WebFetchTool/utils.ts
export function applyPromptToMarkdown(
  content: string,
  prompt: string
): string {
  // 把 prompt 和 content 结合
  // 典型的 prompt = "提取页面中关于 X 的信息"
  // 返回的是处理后的结果，不是原始 HTML
}
```

### 3.5 输出格式

```typescript
// output schema
const outputSchema = z.object({
  bytes: number,           // 内容大小
  code: number,            // HTTP 状态码
  codeText: string,        // HTTP 状态文本
  result: string,          // 处理后的结果（prompt 应用后的内容）
  durationMs: number,      // 耗时
  url: string,             // 实际抓取的 URL
})
```

---

## 四、两者的配合使用

典型工作流：

```
用户：帮我搜索一下 React 最新的新闻
  ↓
Agent 调用 WebSearchTool
  → query: "React latest news"
  → results: [
      {title: "React 19 发布", url: "https://react.dev/blog/react-19"},
      {title: "React Server Components 指南", url: "https://react.dev/guide/rsc"},
      ...
    ]
  ↓
Agent 可能对前几个结果调用 WebFetchTool
  → url: "https://react.dev/blog/react-19"
  → prompt: "总结这篇博客的主要内容"
  → result: "React 19 引入了新特性..."
  ↓
Agent 综合搜索结果 + fetch 的内容回答用户
```

---

## 五、URL 白名单机制

```typescript
// src/tools/WebFetchTool/preapproved.ts
export function isPreapprovedUrl(url: string): boolean {
  try {
    const { hostname } = new URL(url)
    return isPreapprovedHost(hostname)
  } catch {
    return false
  }
}
```

如果 URL 不在白名单里，会触发权限确认（类似文件访问权限），防止 Agent 随意抓取内部页面或陌生网站。

---

## 六、搜索引擎 fallback 设计

```typescript
// backend.ts
export function shouldFallbackFromNativeError(error): boolean {
  // 429 Rate limit → fallback
  // 503 Service unavailable → fallback
  // 其他错误 → 不 fallback
}

export function getFallbackProvider(): string {
  // 获取备用搜索 provider 的配置
}
```

这是一种**弹性降级**设计：主搜索不可用时自动切换到备用方案，不阻塞 Agent 工作。

---

## 七、核心问题（学完应该能回答）

1. WebSearchTool 和 WebFetchTool 的分工是什么？为什么要两个工具？
2. WebFetchTool 怎么提取 HTML 页面的正文内容？
3. URL 白名单机制的作用是什么？
4. WebSearchTool 的 fallback 设计解决了什么问题？
5. WebFetchTool 的 prompt 参数是做什么的？

---

## 八、下一步

Day 5 我们学习如何**自定义工具**：从零实现一个自己的工具，以及 MCP（Model Context Protocol）工具的接入方式。