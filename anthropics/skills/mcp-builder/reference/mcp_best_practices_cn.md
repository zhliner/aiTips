# MCP Server 最佳实践

## 快速参考

### Server 命名
- **Python**：`{service}_mcp`（例如 `slack_mcp`）
- **Node/TypeScript**：`{service}-mcp-server`（例如 `slack-mcp-server`）

### Tool 命名
- 使用 snake_case 并带 service 前缀
- 格式：`{service}_{action}_{resource}`
- 示例：`slack_send_message`、`github_create_issue`

### 响应格式
- 同时支持 JSON 和 Markdown 格式
- JSON 用于程序化处理
- Markdown 用于人类可读性

### 分页
- 始终遵守 `limit` 参数
- 返回 `has_more`、`next_offset`、`total_count`
- 默认 20-50 条

### 传输方式
- **Streamable HTTP**：适用于远程 server、多 client 场景
- **stdio**：适用于本地集成、命令行工具
- 避免使用 SSE（已被 streamable HTTP 取代并弃用）

---

## Server 命名规范

遵循以下标准化命名模式：

**Python**：使用格式 `{service}_mcp`（小写加下划线）
- 示例：`slack_mcp`、`github_mcp`、`jira_mcp`

**Node/TypeScript**：使用格式 `{service}-mcp-server`（小写加连字符）
- 示例：`slack-mcp-server`、`github-mcp-server`、`jira-mcp-server`

名称应当通用、描述所集成的服务、易于从任务描述中推断，且不包含版本号。

---

## Tool 命名与设计

### Tool 命名

1. **使用 snake_case**：`search_users`、`create_project`、`get_channel_info`
2. **包含 service 前缀**：预期你的 MCP server 可能与其他 MCP server 一起使用
   - 使用 `slack_send_message` 而非 `send_message`
   - 使用 `github_create_issue` 而非 `create_issue`
3. **面向动作**：以动词开头（get、list、search、create 等）
4. **具体明确**：避免可能与其他 server 冲突的通用名称

### Tool 设计

- Tool 描述必须精确且无歧义地描述功能
- 描述必须与实际功能精确匹配
- 提供 tool 注解（readOnlyHint、destructiveHint、idempotentHint、openWorldHint）
- 保持 tool 操作的聚焦和原子性

---

## 响应格式

所有返回数据的 tool 都应支持多种格式：

### JSON 格式（`response_format="json"`）
- 机器可读的结构化数据
- 包含所有可用字段和元数据
- 一致的字段名和类型
- 用于程序化处理

### Markdown 格式（`response_format="markdown"`，通常为默认值）
- 人类可读的格式化文本
- 使用标题、列表和格式以提高清晰度
- 将时间戳转换为人类可读格式
- 显示名称并在括号中附带 ID
- 省略冗长的元数据

---

## 分页

对于列出资源的 tool：

- **始终遵守 `limit` 参数**
- **实现分页**：使用 `offset` 或基于游标的分页
- **返回分页元数据**：包含 `has_more`、`next_offset`/`next_cursor`、`total_count`
- **永远不要将所有结果加载到内存中**：对大型数据集尤为重要
- **默认使用合理的限制**：通常 20-50 条

分页响应示例：
```json
{
  "total": 150,
  "count": 20,
  "offset": 0,
  "items": [...],
  "has_more": true,
  "next_offset": 20
}
```

---

## 传输选项

### Streamable HTTP

**最适合**：远程 server、web 服务、多 client 场景

**特点**：
- 基于 HTTP 的双向通信
- 支持多个同时连接的 client
- 可作为 web 服务部署
- 支持 server 到 client 的通知

**使用场景**：
- 同时服务多个 client
- 作为云服务部署
- 与 web 应用集成

### stdio

**最适合**：本地集成、命令行工具

**特点**：
- 标准输入/输出流通信
- 设置简单，无需网络配置
- 作为 client 的子进程运行

**使用场景**：
- 为本地开发环境构建工具
- 与桌面应用集成
- 单用户、单会话场景

**注意**：stdio server 不应向 stdout 输出日志（使用 stderr 记录日志）

### 传输方式选择

| 标准 | stdio | Streamable HTTP |
|------|-------|-----------------|
| **部署** | 本地 | 远程 |
| **Client** | 单个 | 多个 |
| **复杂度** | 低 | 中 |
| **实时性** | 否 | 是 |

---

## 安全最佳实践

### 身份验证与授权

**OAuth 2.1**：
- 使用来自权威机构的证书的安全 OAuth 2.1
- 在处理请求之前验证 access token
- 仅接受专门为你 server 准备的 token

**API Key**：
- 将 API key 存储在环境变量中，永远不要写在代码里
- 在 server 启动时验证 key
- 在身份验证失败时提供清晰的错误消息

### 输入验证

- 清理文件路径以防止目录遍历
- 验证 URL 和外部标识符
- 检查参数大小和范围
- 防止系统调用中的命令注入
- 对所有输入使用 schema 验证（Pydantic/Zod）

### 错误处理

- 不要向 client 暴露内部错误
- 在 server 端记录安全相关的错误
- 提供有帮助但不泄露信息的错误消息
- 在错误后清理资源

### DNS 重绑定保护

对于本地运行的 streamable HTTP server：
- 启用 DNS 重绑定保护
- 验证所有传入连接的 `Origin` header
- 绑定到 `127.0.0.1` 而非 `0.0.0.0`

---

## Tool 注解

提供注解以帮助 client 了解 tool 行为：

| 注解 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `readOnlyHint` | boolean | false | Tool 不修改其环境 |
| `destructiveHint` | boolean | true | Tool 可能执行破坏性更新 |
| `idempotentHint` | boolean | false | 使用相同参数重复调用不会产生额外效果 |
| `openWorldHint` | boolean | true | Tool 与外部实体交互 |

**重要提示**：注解是提示，不是安全保证。Client 不应仅基于注解做出安全关键决策。

---

## 错误处理

- 使用标准 JSON-RPC 错误码
- 在 result 对象中报告 tool 错误（而非协议级错误）
- 提供有帮助的、具体的错误消息及建议的后续步骤
- 不要暴露内部实现细节
- 在错误时正确清理资源

错误处理示例：
```typescript
try {
  const result = performOperation();
  return { content: [{ type: "text", text: result }] };
} catch (error) {
  return {
    isError: true,
    content: [{
      type: "text",
      text: `Error: ${error.message}. Try using filter='active_only' to reduce results.`
    }]
  };
}
```

---

## 测试要求

全面的测试应覆盖：

- **功能测试**：验证使用有效/无效输入时的正确执行
- **集成测试**：测试与外部系统的交互
- **安全测试**：验证身份验证、输入清理、速率限制
- **性能测试**：检查负载下的行为和超时
- **错误处理**：确保正确的错误报告和清理

---

## 文档要求

- 提供所有 tool 和功能的清晰文档
- 包含可运行的示例（每个主要功能至少 3 个）
- 记录安全注意事项
- 指定所需的权限和访问级别
- 记录速率限制和性能特征
