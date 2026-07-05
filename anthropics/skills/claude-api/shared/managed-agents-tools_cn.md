# Managed Agents — 工具与 Skills

## 工具

### 服务端工具 vs 客户端工具

| 类型 | 执行方 | 工作方式 |
|---|---|---|
| **预构建 Claude Agent 工具**（`agent_toolset_20260401`） | Anthropic，在会话的容器中（对于 `cloud` 环境；对于 `self_hosted`，由**你的** worker 提供并运行 —— 参见 `shared/managed-agents-self-hosted-sandboxes.md`） | 文件操作、bash、网页搜索等。一次性全部启用或使用 `enabled: true/false` 单独配置。 |
| **MCP 工具**（`mcp_toolset`） | Anthropic 的编排层 | 已连接的 MCP 服务器暴露的能力。通过 toolset 按服务器授权访问。 |
| **自定义工具** | **你** —— 你的应用处理调用并返回结果 | Agent 发出 `agent.custom_tool_use` 事件，会话进入 `idle`，你发回 `user.custom_tool_result` 事件。 |

**建议：** 通过 `agent_toolset_20260401` 启用所有预构建工具，然后根据需要单独禁用。

**版本控制：** 工具集是一个有版本的静态资源。当底层工具发生变化时，会创建新的工具集版本（因此有 `_20260401`），这样你始终清楚自己使用的具体内容。

### Agent Toolset

`agent_toolset_20260401` 提供以下内置工具：

| 工具                   | 描述                              |
| ---------------------- | ---------------------------------------- |
| `bash` | 在 shell 会话中执行 bash 命令 |
| `read` | 从本地文件系统读取文件，包括文本、图片、PDF 和 Jupyter notebooks |
| `write` | 将文件写入本地文件系统 |
| `edit` | 在文件中执行字符串替换 |
| `glob` | 使用 glob 模式进行快速文件模式匹配 |
| `grep` | 使用正则表达式进行文本搜索 |
| `web_fetch` | 从 URL 获取内容 |
| `web_search` | 搜索网页信息 |

启用完整工具集：

```json
{
  "tools": [
    { "type": "agent_toolset_20260401" }
  ]
}
```

### 单工具配置

覆盖单个工具的默认设置。此示例启用除 bash 外的所有工具：

```json
{
  "tools": [
    {
      "type": "agent_toolset_20260401",
      "default_config": { "enabled": true },
      "configs": [
        { "name": "bash", "enabled": false }
      ]
    }
  ]
}
```

| 字段 | 必填 | 描述 |
|---|---|---|
| `type` | ✅ | `"agent_toolset_20260401"` |
| `default_config` | ❌ | 应用于所有工具。`{ "enabled": bool, "permission_policy": {...} }` |
| `configs` | ❌ | 单工具覆盖：`[{ "name": "...", "enabled": bool, "permission_policy": {...} }]` |

### 权限策略

控制服务端执行的工具（Agent toolset + MCP）是自动运行还是等待审批。不适用于自定义工具。

| 策略 | 行为 |
|---|---|
| `always_allow` | 工具自动执行（默认） |
| `always_ask` | 会话发出 `session.status_idle` 并暂停，直到你发送 `tool_confirmation` 事件 |

```json
{
  "type": "agent_toolset_20260401",
  "default_config": {
    "enabled": true,
    "permission_policy": { "type": "always_allow" }
  },
  "configs": [
    { "name": "bash", "permission_policy": { "type": "always_ask" } }
  ]
}
```

**响应 `always_ask`：** 发送一个 `user.tool_confirmation` 事件，携带触发该事件的 `agent_tool_use`/`mcp_tool_use` 中的 `tool_use_id`：

```json
{ "type": "tool_confirmation", "tool_use_id": "sevt_abc123", "result": "allow" }
{ "type": "tool_confirmation", "tool_use_id": "sevt_def456", "result": "deny", "message": "Read .env.example instead" }
```

deny 时可选的 `message` 会传递给 Agent，以便其调整策略。

要仅启用特定工具，将默认值关闭并按工具逐个开启：

```json
{
  "tools": [
    {
      "type": "agent_toolset_20260401",
      "default_config": { "enabled": false },
      "configs": [
        { "name": "bash", "enabled": true },
        { "name": "read", "enabled": true }
      ]
    }
  ]
}
```

### 自定义工具（客户端）

自定义工具由**你的应用**执行，而非 Anthropic。流程如下：

1. Agent 决定使用该工具 → 会话发出 `agent.custom_tool_use` 事件并携带输入
2. 会话进入 `idle` 等待你
3. 你的应用执行该工具
4. 你发回一个 `user.custom_tool_result` 事件并携带输出
5. 会话恢复为 `running`

无需权限策略 —— 因为你就是执行方。

```json
{
  "tools": [
    {
      "type": "custom",
      "name": "get_weather",
      "description": "Fetch current weather for a city.",
      "input_schema": {
        "type": "object",
        "properties": {
          "city": { "type": "string", "description": "City name" }
        },
        "required": ["city"]
      }
    }
  ]
}
```

### MCP 服务器

MCP（Model Context Protocol）服务器暴露标准化的第三方能力（例如 Asana、GitHub、Linear）。**配置分布在 Agent 和 Vault 两侧：**

1. **Agent 创建**声明要连接哪些服务器（`type`、`name`、`url` —— 不含认证信息）。Agent 的 `mcp_servers` 数组没有 auth 字段。
2. **Vault** 存储 OAuth 凭据。在创建会话时通过 `vault_ids` 附加。

这样可以将密钥排除在可复用的 Agent 定义之外。每个 Vault 凭据绑定到一个 MCP 服务器 URL；Anthropic 通过 URL 将凭据匹配到服务器。

**Agent 侧 —— 声明服务器（不含认证）：**

| 字段 | 必填 | 描述 |
|---|---|---|
| `type` | ✅ | `"url"` |
| `name` | ✅ | 唯一名称 —— 由 `mcp_toolset.mcp_server_name` 引用 |
| `url` | ✅ | MCP 服务器的端点 URL（Streamable HTTP 传输） |

```json
{
  "mcp_servers": [
    { "type": "url", "name": "linear", "url": "https://mcp.linear.app/mcp" }
  ],
  "tools": [
    { "type": "mcp_toolset", "mcp_server_name": "linear" }
  ]
}
```

**会话侧 —— 附加 Vault：**

```json
{
  "agent": "agent_abc123",
  "environment_id": "env_abc123",
  "vault_ids": ["vlt_abc123"]
}
```

> 💡 **按工具启用（经验性）：** 已观察到 `mcp_toolset` 接受 `default_config: {enabled: false}` + `configs: [{name, enabled: true}]` 以实现白名单模式。API 参考仅展示了最小的 `{type, mcp_server_name}` 形式。

> 💡 **在运行中的会话上更改工具/MCP 服务器：** `sessions.update()` 可以在会话处于 `idle` 时替换 `agent.tools`、`agent.mcp_servers` 和 `vault_ids` —— 这是会话级覆盖，不会修改 Agent 对象。参见 `shared/managed-agents-core.md` → Updating the agent configuration mid-session。

**大型 MCP 工具输出。** 如果 MCP 工具返回超过 **100K token** 的输出，输出会自动卸载到沙箱中的文件 —— Agent 收到截断的预览以及文件路径，可以 `read` 完整内容。无需配置。

**无效的 Vault 凭据不会阻止会话创建。** 如果某个 Vault 凭据对已声明的 MCP 服务器无效，会话仍然会成功创建；一个 `session.error` 事件描述 MCP 认证失败，并在下一次 `session.status_idle` → `session.status_running` 转换时重试认证。

> ⚠️ **MCP 认证 token ≠ REST API token。** 托管 MCP 服务器（`mcp.notion.com`、`mcp.linear.app` 等）通常需要 **OAuth bearer token**，而非该服务的原生 API 密钥。Notion 的 `ntn_` 集成 token 可以对 Notion 的 REST API 进行认证，但**不会**作为 Notion MCP 服务器的 Vault 凭据生效。这是两套不同的认证系统。

### Vault —— 凭据存储

**Vault** 存储由 Anthropic 代为管理的凭据。两类凭据：

- **MCP 凭据**（`mcp_oauth`、`static_bearer`）—— 以 `mcp_server_url` 为键。当 Agent 连接到该 URL 的服务器时，token 会自动注入。`mcp_oauth` token 通过标准 OAuth 2.0 `refresh_token` 授权自动刷新。这是认证 MCP 服务器的唯一方式。
- **环境变量**（`environment_variable`）—— 以 `secret_name`（环境变量名）为键。沙箱只看到**不透明的占位符**；真实密钥在出站请求**到达出口时**替换。适用于任何通过环境变量认证的服务：CLI（`aws`、`gcloud`、`stripe`）、SDK 或 `bash` 工具中的直接 `curl` 调用。

你提供的密钥字段（`token`、`access_token`、`refresh_token`、`client_secret`、`secret_value`）为仅写入 —— 永远不会在 API 响应中返回。

#### 凭据与沙箱

Vault 存储凭据；这些凭据**永远不会进入沙箱**。这是一个有意设计的安全边界 —— 在沙箱中运行的代码（包括 Agent 编写的任何内容）无法读取或泄露 Vault 中的凭据，即使在提示注入的情况下也是如此。相反，凭据由 Anthropic 侧代理在请求**离开沙箱后**注入：

- **MCP 工具调用**通过 Anthropic 侧代理路由，该代理从 Vault 获取凭据并添加到出站请求中。
- **对已附加 GitHub 仓库的 Git 操作**（`git pull`、`git push`、GitHub REST 调用）通过 git 代理路由，以相同方式注入 `github_repository` 资源的 `authorization_token`。
- **环境变量凭据**在沙箱中显示为不透明占位符；真实值在出站时替换占位符，且仅限于该凭据允许的主机上的请求。

**当 Vault 凭据不适用时**（例如自托管沙箱 —— `environment_variable` 在那里尚不支持），**请注册自定义工具：** Agent 发出 `agent.custom_tool_use`，你的编排器（已持有凭据）执行调用并通过同一认证事件流返回 `user.custom_tool_result`。无需暴露公共端点；沙箱永远不会看到密钥。参见 `shared/managed-agents-client-patterns.md` → Pattern 9。

**不要将 API 密钥放在系统提示或用户消息中作为变通方案** —— 它们会持久存在于会话的事件历史中。

> 前身为内部称为 TAT（Tool/Tenant Access Tokens）。

**流程：**

1. 创建 Vault（`client.beta.vaults.create(...)`）—— 每个租户/用户一个，或共享一个，取决于你的模型
2. 向其中添加凭据（`client.beta.vaults.credentials.create(...)`）—— MCP 凭据以 MCP 服务器 URL 为键；环境变量凭据以 `secret_name` 为键
3. 在创建会话时通过 `vault_ids: ["vlt_..."]` 引用 Vault
4. Anthropic 在 OAuth token 过期前自动刷新，并在运行时替换密钥

**MCP OAuth 凭据结构**：

```json
{
  "display_name": "Notion (workspace-foo)",
  "auth": {
    "type": "mcp_oauth",
    "mcp_server_url": "https://mcp.notion.com/mcp",
    "access_token": "<current access token>",
    "expires_at": "2026-04-02T14:00:00Z",
    "refresh": {
      "refresh_token": "<refresh token>",
      "client_id": "<your OAuth client_id>",
      "token_endpoint": "https://api.notion.com/v1/oauth/token",
      "token_endpoint_auth": { "type": "none" }
    }
  }
}
```

`refresh` 块实现了自动刷新 —— `token_endpoint` 是 Anthropic 发送 `refresh_token` 授权请求的地址。`token_endpoint_auth` 是一个判别联合类型：

| `type` | 结构 | 使用场景 |
|---|---|---|
| `"none"` | `{type: "none"}` | 公共 OAuth 客户端（无密钥） |
| `"client_secret_basic"` | `{type: "client_secret_basic", client_secret: "..."}` | 机密客户端，通过 HTTP Basic 认证传递密钥 |
| `"client_secret_post"` | `{type: "client_secret_post", client_secret: "..."}` | 机密客户端，在请求体中传递密钥 |

如果你只有 access token 而没有刷新能力，可以完全省略 `refresh` —— 它会在过期前正常工作，之后 Agent 将失去访问权限。

> 💡 **获取 OAuth token。** 如何获取初始的 access token 和 refresh token 取决于 MCP 服务器 —— 请查阅其文档。获取后，使用上述结构将其存储在 Vault 凭据中；Anthropic 会通过 `refresh.token_endpoint` 自动刷新。

**环境变量凭据结构**：

```json
{
  "display_name": "Twilio API key for sandbox",
  "auth": {
    "type": "environment_variable",
    "secret_name": "TWILIO_API_KEY",
    "secret_value": "sk-your-secret-here",
    "networking": {
      "type": "limited",
      "allowed_hosts": ["api.twilio.com", "*.twilio.com"]
    }
  }
}
```

`networking.allowed_hosts` 控制密钥可以替换到哪些出站主机 —— `{"type": "limited", "allowed_hosts": [...]}` 或 `{"type": "unrestricted"}`（如果你无法预先枚举域名）。强烈建议限制：它可以防止密钥被发送到未授权的主机。

**`injection_location`**（可选，`networking` 的同级字段）控制密钥在出站请求中替换的**位置** —— `{header: bool, body: bool}`。两者相互独立：`allowed_hosts` 限定*哪些主机*可以被替换后的请求访问；`injection_location` 限定*请求的哪些部分*在所有这些主机上会被替换密钥。大多数服务从请求头读取 API 密钥，因此 `{"header": true}` 是更严格的配置 —— 请求体通常由 Agent 正在处理的内容组装而成，使 body 成为更大的暴露面。在禁用位置的占位符**既不会被替换也不会被移除** —— 字面的不透明占位符字符串会在该位置发送给第三方。

| 操作 | `injection_location` 语义 |
|---|---|
| 创建凭据 | 完全省略该字段 → 两个位置均启用。提供该对象 → 省略的任何字段默认为 `false`（`{"header": true}` 创建仅 header 的凭据）。 |
| 更新凭据 | 字段**单独合并** —— `{"body": false}` 禁用 body 替换并保持 `header` 不变。对于运行中的会话，更新在会话的下次操作时生效。 |

凭据必须至少启用一个位置；创建或更新操作如果会禁用两者则返回 400，显式 `null` 的对象或任一字段同样返回 400（请改为省略）。响应始终返回两个字段及其解析后的值。

> ⚠️ **两个网络层，缺一不可。** 凭据上的 `networking.allowed_hosts` 控制哪些请求*使用密钥*，而非哪些请求被*允许*。Agent 还必须能够在**环境层**访问该域名（`unrestricted`，或环境的 `allowed_hosts` 中列出的主机 —— 参见 `shared/managed-agents-environments.md`）。任一层缺少域名都会导致密钥替换后的请求失败。

> ⚠️ **客户端验证注意事项。** 替换发生在出站时，而非沙箱内部 —— 在发起网络请求前*本地*验证凭据*格式*的客户端（例如检查密钥以 `sk-` 开头的 CLI）会看到不透明占位符并可能在启动时失败。如果客户端在任何网络调用之前就拒绝了凭据，这就是原因。

> 💡 **最小化密钥权限范围。** Agent 可以执行密钥允许的任何操作；如果密钥的权限超出任务所需，会在 Agent 行为异常时增加影响范围。

**自托管沙箱不支持** —— `environment_variable` 凭据需要 Anthropic 管理的出站。参见 `shared/managed-agents-self-hosted-sandboxes.md`。

**约束（所有凭据类型）：**

- **每个 Vault 唯一键。** `mcp_server_url`（MCP 凭据）和 `secret_name`（环境变量凭据）在 Vault 的活动凭据中必须唯一；重复返回 409。
- **键不可变。** 密钥值、`display_name` 以及（环境变量凭据的）`injection_location` 可以更新；要更改 `mcp_server_url`、`secret_name`、`token_endpoint` 或 `client_id`，请归档凭据并创建新的。归档会清除密钥并释放键以供替换。
- **每个 Vault 最多 20 个凭据。**
- 凭据按原样存储，**在会话运行时之前不会验证** —— 无效凭据会在会话期间表现为认证或下游错误，该错误会被发出但不会阻止会话继续。

**作用域：** Vault 为工作空间级。API 工作空间中具有 developer+ 角色的任何人都可以创建、读取（仅元数据 —— 密钥为仅写入）和附加 Vault。`vault_ids` 可以在会话**创建**时设置，但不能通过会话更新设置（SDK 文档字符串注明 "Not yet supported; requests setting this field are rejected"）。

---

## Skills

Skills 是可复用的、基于文件系统的资源，为你的 Agent 提供领域专业知识：工作流、上下文和最佳实践，将通用 Agent 转变为专家。与提示（针对一次性任务的会话级指令）不同，Skills 按需加载，无需在多个对话中反复提供相同的指导。

两种类型 —— 工作方式相同；Agent 会在与当前任务相关时自动使用它们：

| 类型 | 说明 |
|---|---|
| **预构建 Anthropic Skills** | 常见文档任务（PowerPoint、Excel、Word、PDF）。按名称引用（例如 `xlsx`）。 |
| **自定义 Skills** | 你通过 Skills API 在组织中创建的 Skills。通过 `skill_id` + 可选 `version` 引用。 |

**每个 Agent 最多 20 个 Skills。** Agent 创建使用 `managed-agents-2026-04-01`；独立的 Skills API（用于管理自定义 Skill 定义）使用 `skills-2025-10-02`。

### 在会话上启用 Skills

Skills 通过 `agents.create()` 附加到 **Agent** 定义：

```ts
const agent = await client.beta.agents.create(
  {
    name: "Financial Agent",
    model: "claude-opus-4-8",
    system: "You are a financial analysis agent.",
    skills: [
      { type: "anthropic", skill_id: "xlsx" },
      { type: "custom", skill_id: "skill_abc123", version: "latest" },
    ],
  }
);
```

Python：

```python
agent = client.beta.agents.create(
    name="Financial Agent",
    model="claude-opus-4-8",
    system="You are a financial analysis agent.",
    skills=[
        {"type": "anthropic", "skill_id": "xlsx"},
        {"type": "custom", "skill_id": "skill_abc123", "version": "latest"},
    ]
)
```

**Skill 引用字段：**

| 字段 | Anthropic Skill | 自定义 Skill |
|---|---|---|
| `type` | `"anthropic"` | `"custom"` |
| `skill_id` | Skill 名称（例如 `"xlsx"`、`"docx"`、`"pptx"`、`"pdf"`） | Skills API 中的 Skill ID（例如 `"skill_abc123"`） |
| `version` | — | `"latest"` 或特定版本号 |

### Skills API

| 操作             | 方法   | 路径                                            |
| --------------------- | -------- | ----------------------------------------------- |
| 创建 Skill          | `POST`   | `/v1/skills`                                    |
| 列出 Skills           | `GET`    | `/v1/skills`                                    |
| 获取 Skill             | `GET`    | `/v1/skills/{id}`                               |
| 删除 Skill          | `DELETE` | `/v1/skills/{id}`                               |
| 创建版本        | `POST`   | `/v1/skills/{id}/versions`                      |
| 列出版本         | `GET`    | `/v1/skills/{id}/versions`                      |
| 获取版本           | `GET`    | `/v1/skills/{id}/versions/{version}`            |
| 删除版本        | `DELETE` | `/v1/skills/{id}/versions/{version}`            |