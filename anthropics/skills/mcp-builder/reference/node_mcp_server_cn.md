# Node/TypeScript MCP Server 实现指南

## 概述

本文档提供使用 MCP TypeScript SDK 实现 MCP server 的 Node/TypeScript 特定最佳实践和示例。涵盖项目结构、server 设置、tool 注册模式、使用 Zod 进行输入验证、错误处理以及完整的可运行示例。

---

## 快速参考

### 关键导入
```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import express from "express";
import { z } from "zod";
```

### Server 初始化
```typescript
const server = new McpServer({
  name: "service-mcp-server",
  version: "1.0.0"
});
```

### Tool 注册模式
```typescript
server.registerTool(
  "tool_name",
  {
    title: "Tool Display Name",
    description: "What the tool does",
    inputSchema: { param: z.string() },
    outputSchema: { result: z.string() }
  },
  async ({ param }) => {
    const output = { result: `Processed: ${param}` };
    return {
      content: [{ type: "text", text: JSON.stringify(output) }],
      structuredContent: output // 结构化数据的现代模式
    };
  }
);
```

---

## MCP TypeScript SDK

官方 MCP TypeScript SDK 提供：
- `McpServer` 类用于 server 初始化
- `registerTool` 方法用于 tool 注册
- Zod schema 集成用于运行时输入验证
- 类型安全的 tool handler 实现

**重要 - 仅使用现代 API：**
- **应使用**：`server.registerTool()`、`server.registerResource()`、`server.registerPrompt()`
- **不应使用**：旧的已弃用 API，如 `server.tool()`、`server.setRequestHandler(ListToolsRequestSchema, ...)` 或手动 handler 注册
- `register*` 方法提供更好的类型安全性、自动 schema 处理，是推荐的方式

完整详情请参阅参考文档中的 MCP SDK 文档。

## Server 命名规范

Node/TypeScript MCP server 必须遵循以下命名模式：
- **格式**：`{service}-mcp-server`（小写加连字符）
- **示例**：`github-mcp-server`、`jira-mcp-server`、`stripe-mcp-server`

名称应当：
- 通用（不与特定功能绑定）
- 描述所集成的服务/API
- 易于从任务描述中推断
- 不包含版本号或日期

## 项目结构

为 Node/TypeScript MCP server 创建以下结构：

```
{service}-mcp-server/
├── package.json
├── tsconfig.json
├── README.md
├── src/
│   ├── index.ts          # 主入口，包含 McpServer 初始化
│   ├── types.ts          # TypeScript 类型定义和接口
│   ├── tools/            # Tool 实现（每个领域一个文件）
│   ├── services/         # API client 和共享工具
│   ├── schemas/          # Zod 验证 schema
│   └── constants.ts      # 共享常量（API_URL、CHARACTER_LIMIT 等）
└── dist/                 # 构建后的 JavaScript 文件（入口：dist/index.js）
```

## Tool 实现

### Tool 命名

使用 snake_case 作为 tool 名称（例如 "search_users"、"create_project"、"get_channel_info"），名称应清晰且面向动作。

**避免命名冲突**：包含 service 上下文以防止重叠：
- 使用 "slack_send_message" 而非 "send_message"
- 使用 "github_create_issue" 而非 "create_issue"
- 使用 "asana_list_tasks" 而非 "list_tasks"

### Tool 结构

Tool 使用 `registerTool` 方法注册，需满足以下要求：
- 使用 Zod schema 进行运行时输入验证和类型安全
- `description` 字段必须显式提供 - JSDoc 注释不会被自动提取
- 显式提供 `title`、`description`、`inputSchema` 和 `annotations`
- `inputSchema` 必须是 Zod schema 对象（不是 JSON schema）
- 显式标注所有参数和返回值的类型

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";

const server = new McpServer({
  name: "example-mcp",
  version: "1.0.0"
});

// 用于输入验证的 Zod schema
const UserSearchInputSchema = z.object({
  query: z.string()
    .min(2, "Query must be at least 2 characters")
    .max(200, "Query must not exceed 200 characters")
    .describe("Search string to match against names/emails"),
  limit: z.number()
    .int()
    .min(1)
    .max(100)
    .default(20)
    .describe("Maximum results to return"),
  offset: z.number()
    .int()
    .min(0)
    .default(0)
    .describe("Number of results to skip for pagination"),
  response_format: z.nativeEnum(ResponseFormat)
    .default(ResponseFormat.MARKDOWN)
    .describe("Output format: 'markdown' for human-readable or 'json' for machine-readable")
}).strict();

// 从 Zod schema 定义类型
type UserSearchInput = z.infer<typeof UserSearchInputSchema>;

server.registerTool(
  "example_search_users",
  {
    title: "Search Example Users",
    description: `Search for users in the Example system by name, email, or team.

This tool searches across all user profiles in the Example platform, supporting partial matches and various search filters. It does NOT create or modify users, only searches existing ones.

Args:
  - query (string): Search string to match against names/emails
  - limit (number): Maximum results to return, between 1-100 (default: 20)
  - offset (number): Number of results to skip for pagination (default: 0)
  - response_format ('markdown' | 'json'): Output format (default: 'markdown')

Returns:
  For JSON format: Structured data with schema:
  {
    "total": number,           // 找到的匹配总数
    "count": number,           // 本次响应中的结果数
    "offset": number,          // 当前分页偏移量
    "users": [
      {
        "id": string,          // 用户 ID（例如 "U123456789"）
        "name": string,        // 全名（例如 "John Doe"）
        "email": string,       // 电子邮件地址
        "team": string,        // 团队名称（可选）
        "active": boolean      // 用户是否活跃
      }
    ],
    "has_more": boolean,       // 是否有更多结果
    "next_offset": number      // 下一页的偏移量（如果 has_more 为 true）
  }

Examples:
  - Use when: "Find all marketing team members" -> params with query="team:marketing"
  - Use when: "Search for John's account" -> params with query="john"
  - Don't use when: You need to create a user (use example_create_user instead)

Error Handling:
  - Returns "Error: Rate limit exceeded" if too many requests (429 status)
  - Returns "No users found matching '<query>'" if search returns empty`,
    inputSchema: UserSearchInputSchema,
    annotations: {
      readOnlyHint: true,
      destructiveHint: false,
      idempotentHint: true,
      openWorldHint: true
    }
  },
  async (params: UserSearchInput) => {
    try {
      // 输入验证由 Zod schema 处理
      // 使用验证后的参数发起 API 请求
      const data = await makeApiRequest<any>(
        "users/search",
        "GET",
        undefined,
        {
          q: params.query,
          limit: params.limit,
          offset: params.offset
        }
      );

      const users = data.users || [];
      const total = data.total || 0;

      if (!users.length) {
        return {
          content: [{
            type: "text",
            text: `No users found matching '${params.query}'`
          }]
        };
      }

      // 准备结构化输出
      const output = {
        total,
        count: users.length,
        offset: params.offset,
        users: users.map((user: any) => ({
          id: user.id,
          name: user.name,
          email: user.email,
          ...(user.team ? { team: user.team } : {}),
          active: user.active ?? true
        })),
        has_more: total > params.offset + users.length,
        ...(total > params.offset + users.length ? {
          next_offset: params.offset + users.length
        } : {})
      };

      // 根据请求格式生成文本表示
      let textContent: string;
      if (params.response_format === ResponseFormat.MARKDOWN) {
        const lines = [`# User Search Results: '${params.query}'`, "",
          `Found ${total} users (showing ${users.length})`, ""];
        for (const user of users) {
          lines.push(`## ${user.name} (${user.id})`);
          lines.push(`- **Email**: ${user.email}`);
          if (user.team) lines.push(`- **Team**: ${user.team}`);
          lines.push("");
        }
        textContent = lines.join("\n");
      } else {
        textContent = JSON.stringify(output, null, 2);
      }

      return {
        content: [{ type: "text", text: textContent }],
        structuredContent: output // 结构化数据的现代模式
      };
    } catch (error) {
      return {
        content: [{
          type: "text",
          text: handleApiError(error)
        }]
      };
    }
  }
);
```

## 使用 Zod 进行输入验证

Zod 提供运行时类型验证：

```typescript
import { z } from "zod";

// 带验证的基础 schema
const CreateUserSchema = z.object({
  name: z.string()
    .min(1, "Name is required")
    .max(100, "Name must not exceed 100 characters"),
  email: z.string()
    .email("Invalid email format"),
  age: z.number()
    .int("Age must be a whole number")
    .min(0, "Age cannot be negative")
    .max(150, "Age cannot be greater than 150")
}).strict();  // 使用 .strict() 禁止额外字段

// 枚举
enum ResponseFormat {
  MARKDOWN = "markdown",
  JSON = "json"
}

const SearchSchema = z.object({
  response_format: z.nativeEnum(ResponseFormat)
    .default(ResponseFormat.MARKDOWN)
    .describe("Output format")
});

// 带默认值的可选字段
const PaginationSchema = z.object({
  limit: z.number()
    .int()
    .min(1)
    .max(100)
    .default(20)
    .describe("Maximum results to return"),
  offset: z.number()
    .int()
    .min(0)
    .default(0)
    .describe("Number of results to skip")
});
```

## 响应格式选项

支持多种输出格式以提高灵活性：

```typescript
enum ResponseFormat {
  MARKDOWN = "markdown",
  JSON = "json"
}

const inputSchema = z.object({
  query: z.string(),
  response_format: z.nativeEnum(ResponseFormat)
    .default(ResponseFormat.MARKDOWN)
    .describe("Output format: 'markdown' for human-readable or 'json' for machine-readable")
});
```

**Markdown 格式**：
- 使用标题、列表和格式以提高清晰度
- 将时间戳转换为人类可读格式
- 显示名称并在括号中附带 ID
- 省略冗长的元数据
- 逻辑分组相关信息

**JSON 格式**：
- 返回完整的、适合程序化处理的结构化数据
- 包含所有可用字段和元数据
- 使用一致的字段名和类型

## 分页实现

对于列出资源的 tool：

```typescript
const ListSchema = z.object({
  limit: z.number().int().min(1).max(100).default(20),
  offset: z.number().int().min(0).default(0)
});

async function listItems(params: z.infer<typeof ListSchema>) {
  const data = await apiRequest(params.limit, params.offset);

  const response = {
    total: data.total,
    count: data.items.length,
    offset: params.offset,
    items: data.items,
    has_more: data.total > params.offset + data.items.length,
    next_offset: data.total > params.offset + data.items.length
      ? params.offset + data.items.length
      : undefined
  };

  return JSON.stringify(response, null, 2);
}
```

## 字符限制与截断

添加 CHARACTER_LIMIT 常量以防止响应过大：

```typescript
// 在 constants.ts 的模块级别
export const CHARACTER_LIMIT = 25000;  // 最大响应字符数

async function searchTool(params: SearchInput) {
  let result = generateResponse(data);

  // 检查字符限制，必要时截断
  if (result.length > CHARACTER_LIMIT) {
    const truncatedData = data.slice(0, Math.max(1, data.length / 2));
    response.data = truncatedData;
    response.truncated = true;
    response.truncation_message =
      `Response truncated from ${data.length} to ${truncatedData.length} items. ` +
      `Use 'offset' parameter or add filters to see more results.`;
    result = JSON.stringify(response, null, 2);
  }

  return result;
}
```

## 错误处理

提供清晰、可操作的错误消息：

```typescript
import axios, { AxiosError } from "axios";

function handleApiError(error: unknown): string {
  if (error instanceof AxiosError) {
    if (error.response) {
      switch (error.response.status) {
        case 404:
          return "Error: Resource not found. Please check the ID is correct.";
        case 403:
          return "Error: Permission denied. You don't have access to this resource.";
        case 429:
          return "Error: Rate limit exceeded. Please wait before making more requests.";
        default:
          return `Error: API request failed with status ${error.response.status}`;
      }
    } else if (error.code === "ECONNABORTED") {
      return "Error: Request timed out. Please try again.";
    }
  }
  return `Error: Unexpected error occurred: ${error instanceof Error ? error.message : String(error)}`;
}
```

## 共享工具

将通用功能提取为可复用的函数：

```typescript
// 共享 API 请求函数
async function makeApiRequest<T>(
  endpoint: string,
  method: "GET" | "POST" | "PUT" | "DELETE" = "GET",
  data?: any,
  params?: any
): Promise<T> {
  try {
    const response = await axios({
      method,
      url: `${API_BASE_URL}/${endpoint}`,
      data,
      params,
      timeout: 30000,
      headers: {
        "Content-Type": "application/json",
        "Accept": "application/json"
      }
    });
    return response.data;
  } catch (error) {
    throw error;
  }
}
```

## Async/Await 最佳实践

始终使用 async/await 处理网络请求和 I/O 操作：

```typescript
// 好的做法：异步网络请求
async function fetchData(resourceId: string): Promise<ResourceData> {
  const response = await axios.get(`${API_URL}/resource/${resourceId}`);
  return response.data;
}

// 不好的做法：Promise 链
function fetchData(resourceId: string): Promise<ResourceData> {
  return axios.get(`${API_URL}/resource/${resourceId}`)
    .then(response => response.data);  // 更难阅读和维护
}
```

## TypeScript 最佳实践

1. **使用严格 TypeScript**：在 tsconfig.json 中启用严格模式
2. **定义接口**：为所有数据结构创建清晰的接口定义
3. **避免 `any`**：使用适当的类型或 `unknown` 代替 `any`
4. **Zod 用于运行时验证**：使用 Zod schema 验证外部数据
5. **类型守卫**：为复杂类型检查创建类型守卫函数
6. **错误处理**：始终使用 try-catch 并进行适当的错误类型检查
7. **空值安全**：使用可选链（`?.`）和空值合并（`??`）

```typescript
// 好的做法：使用 Zod 和接口的类型安全实现
interface UserResponse {
  id: string;
  name: string;
  email: string;
  team?: string;
  active: boolean;
}

const UserSchema = z.object({
  id: z.string(),
  name: z.string(),
  email: z.string().email(),
  team: z.string().optional(),
  active: z.boolean()
});

type User = z.infer<typeof UserSchema>;

async function getUser(id: string): Promise<User> {
  const data = await apiCall(`/users/${id}`);
  return UserSchema.parse(data);  // 运行时验证
}

// 不好的做法：使用 any
async function getUser(id: string): Promise<any> {
  return await apiCall(`/users/${id}`);  // 没有类型安全
}
```

## 包配置

### package.json

```json
{
  "name": "{service}-mcp-server",
  "version": "1.0.0",
  "description": "MCP server for {Service} API integration",
  "type": "module",
  "main": "dist/index.js",
  "scripts": {
    "start": "node dist/index.js",
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "clean": "rm -rf dist"
  },
  "engines": {
    "node": ">=18"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.6.1",
    "axios": "^1.7.9",
    "zod": "^3.23.8"
  },
  "devDependencies": {
    "@types/node": "^22.10.0",
    "tsx": "^4.19.2",
    "typescript": "^5.7.2"
  }
}
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "allowSyntheticDefaultImports": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

## 完整示例

```typescript
#!/usr/bin/env node
/**
 * Example Service 的 MCP Server。
 *
 * 本 server 提供与 Example API 交互的 tool，包括用户搜索、
 * 项目管理和数据导出功能。
 */

import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";
import axios, { AxiosError } from "axios";

// 常量
const API_BASE_URL = "https://api.example.com/v1";
const CHARACTER_LIMIT = 25000;

// 枚举
enum ResponseFormat {
  MARKDOWN = "markdown",
  JSON = "json"
}

// Zod schema
const UserSearchInputSchema = z.object({
  query: z.string()
    .min(2, "Query must be at least 2 characters")
    .max(200, "Query must not exceed 200 characters")
    .describe("Search string to match against names/emails"),
  limit: z.number()
    .int()
    .min(1)
    .max(100)
    .default(20)
    .describe("Maximum results to return"),
  offset: z.number()
    .int()
    .min(0)
    .default(0)
    .describe("Number of results to skip for pagination"),
  response_format: z.nativeEnum(ResponseFormat)
    .default(ResponseFormat.MARKDOWN)
    .describe("Output format: 'markdown' for human-readable or 'json' for machine-readable")
}).strict();

type UserSearchInput = z.infer<typeof UserSearchInputSchema>;

// 共享工具函数
async function makeApiRequest<T>(
  endpoint: string,
  method: "GET" | "POST" | "PUT" | "DELETE" = "GET",
  data?: any,
  params?: any
): Promise<T> {
  try {
    const response = await axios({
      method,
      url: `${API_BASE_URL}/${endpoint}`,
      data,
      params,
      timeout: 30000,
      headers: {
        "Content-Type": "application/json",
        "Accept": "application/json"
      }
    });
    return response.data;
  } catch (error) {
    throw error;
  }
}

function handleApiError(error: unknown): string {
  if (error instanceof AxiosError) {
    if (error.response) {
      switch (error.response.status) {
        case 404:
          return "Error: Resource not found. Please check the ID is correct.";
        case 403:
          return "Error: Permission denied. You don't have access to this resource.";
        case 429:
          return "Error: Rate limit exceeded. Please wait before making more requests.";
        default:
          return `Error: API request failed with status ${error.response.status}`;
      }
    } else if (error.code === "ECONNABORTED") {
      return "Error: Request timed out. Please try again.";
    }
  }
  return `Error: Unexpected error occurred: ${error instanceof Error ? error.message : String(error)}`;
}

// 创建 MCP server 实例
const server = new McpServer({
  name: "example-mcp",
  version: "1.0.0"
});

// 注册 tool
server.registerTool(
  "example_search_users",
  {
    title: "Search Example Users",
    description: `[Full description as shown above]`,
    inputSchema: UserSearchInputSchema,
    annotations: {
      readOnlyHint: true,
      destructiveHint: false,
      idempotentHint: true,
      openWorldHint: true
    }
  },
  async (params: UserSearchInput) => {
    // 实现如上方所示
  }
);

// 主函数
// 用于 stdio（本地）：
async function runStdio() {
  if (!process.env.EXAMPLE_API_KEY) {
    console.error("ERROR: EXAMPLE_API_KEY environment variable is required");
    process.exit(1);
  }

  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("MCP server running via stdio");
}

// 用于 streamable HTTP（远程）：
async function runHTTP() {
  if (!process.env.EXAMPLE_API_KEY) {
    console.error("ERROR: EXAMPLE_API_KEY environment variable is required");
    process.exit(1);
  }

  const app = express();
  app.use(express.json());

  app.post('/mcp', async (req, res) => {
    const transport = new StreamableHTTPServerTransport({
      sessionIdGenerator: undefined,
      enableJsonResponse: true
    });
    res.on('close', () => transport.close());
    await server.connect(transport);
    await transport.handleRequest(req, res, req.body);
  });

  const port = parseInt(process.env.PORT || '3000');
  app.listen(port, () => {
    console.error(`MCP server running on http://localhost:${port}/mcp`);
  });
}

// 根据环境选择传输方式
const transport = process.env.TRANSPORT || 'stdio';
if (transport === 'http') {
  runHTTP().catch(error => {
    console.error("Server error:", error);
    process.exit(1);
  });
} else {
  runStdio().catch(error => {
    console.error("Server error:", error);
    process.exit(1);
  });
}
```

---

## 高级 MCP 功能

### Resource 注册

将数据作为 resource 暴露，实现高效的、基于 URI 的访问：

```typescript
import { ResourceTemplate } from "@modelcontextprotocol/sdk/types.js";

// 使用 URI 模板注册 resource
server.registerResource(
  {
    uri: "file://documents/{name}",
    name: "Document Resource",
    description: "Access documents by name",
    mimeType: "text/plain"
  },
  async (uri: string) => {
    // 从 URI 中提取参数
    const match = uri.match(/^file:\/\/documents\/(.+)$/);
    if (!match) {
      throw new Error("Invalid URI format");
    }

    const documentName = match[1];
    const content = await loadDocument(documentName);

    return {
      contents: [{
        uri,
        mimeType: "text/plain",
        text: content
      }]
    };
  }
);

// 动态列出可用 resource
server.registerResourceList(async () => {
  const documents = await getAvailableDocuments();
  return {
    resources: documents.map(doc => ({
      uri: `file://documents/${doc.name}`,
      name: doc.name,
      mimeType: "text/plain",
      description: doc.description
    }))
  };
});
```

**何时使用 Resource vs Tool：**
- **Resource**：用于带简单 URI 参数的数据访问
- **Tool**：用于需要验证和业务逻辑的复杂操作
- **Resource**：当数据相对静态或基于模板时
- **Tool**：当操作有副作用或复杂工作流时

### 传输选项

TypeScript SDK 支持两种主要传输机制：

#### Streamable HTTP（推荐用于远程 Server）

```typescript
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import express from "express";

const app = express();
app.use(express.json());

app.post('/mcp', async (req, res) => {
  // 为每个请求创建新的 transport（无状态，防止请求 ID 冲突）
  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: undefined,
    enableJsonResponse: true
  });

  res.on('close', () => transport.close());

  await server.connect(transport);
  await transport.handleRequest(req, res, req.body);
});

app.listen(3000);
```

#### stdio（用于本地集成）

```typescript
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const transport = new StdioServerTransport();
await server.connect(transport);
```

**传输方式选择：**
- **Streamable HTTP**：web 服务、远程访问、多 client
- **stdio**：命令行工具、本地开发、子进程集成

### 通知支持

在 server 状态变化时通知 client：

```typescript
// 当 tool 列表变化时通知
server.notification({
  method: "notifications/tools/list_changed"
});

// 当 resource 变化时通知
server.notification({
  method: "notifications/resources/list_changed"
});
```

谨慎使用通知——仅在 server 能力真正发生变化时使用。

---

## 代码最佳实践

### 代码可组合性与可复用性

你的实现必须优先考虑可组合性和代码复用：

1. **提取通用功能**：
   - 为多个 tool 使用的操作创建可复用的辅助函数
   - 构建共享的 API client 用于 HTTP 请求，而非重复代码
   - 将错误处理逻辑集中在工具函数中
   - 将业务逻辑提取为可组合的专用函数
   - 提取共享的 markdown 或 JSON 字段选择和格式化功能

2. **避免重复**：
   - 永远不要在 tool 之间复制粘贴相似代码
   - 如果发现自己在写相似的逻辑两次，将其提取为函数
   - 分页、过滤、字段选择和格式化等通用操作应当共享
   - 身份验证/授权逻辑应当集中

## 构建与运行

始终在运行之前构建 TypeScript 代码：

```bash
# 构建项目
npm run build

# 运行 server
npm start

# 开发模式（自动重载）
npm run dev
```

始终确保 `npm run build` 成功完成后再认为实现完成。

## 质量检查清单

在完成 Node/TypeScript MCP server 实现之前，确保：

### 战略设计
- [ ] Tool 实现完整的工作流，而不仅仅是 API endpoint 的封装
- [ ] Tool 名称反映自然的任务划分
- [ ] 响应格式针对 agent 上下文效率进行优化
- [ ] 在适当的地方使用人类可读的标识符
- [ ] 错误消息引导 agent 正确使用

### 实现质量
- [ ] 聚焦实现：已实现最重要和最有价值的 tool
- [ ] 所有 tool 使用 `registerTool` 注册并包含完整配置
- [ ] 所有 tool 包含 `title`、`description`、`inputSchema` 和 `annotations`
- [ ] 注解正确设置（readOnlyHint、destructiveHint、idempotentHint、openWorldHint）
- [ ] 所有 tool 使用 Zod schema 进行运行时输入验证并启用 `.strict()` 强制
- [ ] 所有 Zod schema 具有适当的约束和描述性错误消息
- [ ] 所有 tool 具有包含显式输入/输出类型的全面描述
- [ ] 描述包含返回值示例和完整的 schema 文档
- [ ] 错误消息清晰、可操作且具有教育意义

### TypeScript 质量
- [ ] 为所有数据结构定义了 TypeScript 接口
- [ ] 在 tsconfig.json 中启用了严格 TypeScript
- [ ] 未使用 `any` 类型 - 使用 `unknown` 或适当的类型代替
- [ ] 所有 async 函数具有显式的 Promise<T> 返回类型
- [ ] 错误处理使用适当的类型守卫（例如 `axios.isAxiosError`、`z.ZodError`）

### 高级功能（如适用）
- [ ] 为适当的数据 endpoint 注册了 resource
- [ ] 配置了适当的传输方式（stdio 或 streamable HTTP）
- [ ] 为动态 server 能力实现了通知
- [ ] 使用 SDK 接口实现类型安全

### 项目配置
- [ ] Package.json 包含所有必要的依赖
- [ ] 构建脚本在 dist/ 目录中生成可工作的 JavaScript
- [ ] 主入口正确配置为 dist/index.js
- [ ] Server 名称遵循格式：`{service}-mcp-server`
- [ ] tsconfig.json 正确配置了严格模式

### 代码质量
- [ ] 在适用场景下正确实现了分页
- [ ] 大型响应检查 CHARACTER_LIMIT 常量并使用清晰消息截断
- [ ] 为可能的大型结果集提供了过滤选项
- [ ] 所有网络操作优雅地处理超时和连接错误
- [ ] 通用功能已提取为可复用函数
- [ ] 相似操作的返回类型一致

### 测试与构建
- [ ] `npm run build` 成功完成且无错误
- [ ] dist/index.js 已创建并可执行
- [ ] Server 可运行：`node dist/index.js --help`
- [ ] 所有导入正确解析
- [ ] 示例 tool 调用按预期工作
