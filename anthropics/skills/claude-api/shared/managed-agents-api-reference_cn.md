# Managed Agents — 端点参考

所有端点都需要 `x-api-key` 和 `anthropic-version: 2023-06-01` 请求头。Managed Agents 端点还需要 `anthropic-beta` 请求头。

## Beta 请求头

```
anthropic-beta: managed-agents-2026-04-01
```

SDK 会为所有 `client.beta.{agents,environments,sessions,vaults,memory_stores,deployments,deployment_runs}.*` 调用自动添加此请求头。Skills 端点使用 `skills-2025-10-02`；Files 端点使用 `files-api-2025-04-14`。

---

## SDK 方法参考

所有资源都在 `beta` 命名空间下。Python 和 TypeScript 使用相同的方法名。

| 资源 | Python / TypeScript (`client.beta.*`) | Go (`client.Beta.*`) |
| --- | --- | --- |
| Agents | `agents.create` / `retrieve` / `update` / `list` / `archive` | `Agents.New` / `Get` / `Update` / `List` / `Archive` |
| Agent Versions | `agents.versions.list` | `Agents.Versions.List` |
| Environments | `environments.create` / `retrieve` / `update` / `list` / `delete` / `archive` | `Environments.New` / `Get` / `Update` / `List` / `Delete` / `Archive` |
| Environment Work（自托管） | `environments.work.poller` / `stats` / `stop` | 参见 `shared/managed-agents-self-hosted-sandboxes.md` |
| Sessions | `sessions.create` / `retrieve` / `update` / `list` / `delete` / `archive` | `Sessions.New` / `Get` / `Update` / `List` / `Delete` / `Archive` |
| Session Events | `sessions.events.list` / `send` / `stream` | `Sessions.Events.List` / `Send` / `StreamEvents` |
| Session Threads | `sessions.threads.list` / `retrieve` / `archive`; `sessions.threads.events.list` / `stream` | `Sessions.Threads.List` / `Get` / `Archive`; `Sessions.Threads.Events.List` / `StreamEvents` |
| Session Resources | `sessions.resources.add` / `retrieve` / `update` / `list` / `delete` | `Sessions.Resources.Add` / `Get` / `Update` / `List` / `Delete` |
| Deployments | `deployments.create` / `pause` / `unpause` / `archive` / `run` | 尚未文档化 — WebFetch SDK 仓库（`shared/live-sources.md`） |
| Deployment Runs | `deployment_runs.list` / `retrieve`（TS: `deploymentRuns.*`） | 尚未文档化 — WebFetch SDK 仓库（`shared/live-sources.md`） |
| Vaults | `vaults.create` / `retrieve` / `update` / `list` / `delete` / `archive` | `Vaults.New` / `Get` / `Update` / `List` / `Delete` / `Archive` |
| Credentials | `vaults.credentials.create` / `retrieve` / `update` / `list` / `delete` / `archive` / `mcp_oauth_validate` | `Vaults.Credentials.New` / `Get` / `Update` / `List` / `Delete` / `Archive` / `McpOauthValidate` |
| Memory Stores | `memory_stores.create` / `retrieve` / `update` / `list` / `delete` / `archive` | `MemoryStores.New` / `Get` / `Update` / `List` / `Delete` / `Archive` |
| Memories | `memory_stores.memories.create` / `retrieve` / `update` / `list` / `delete` | `MemoryStores.Memories.New` / `Get` / `Update` / `List` / `Delete` |
| Memory Versions | `memory_stores.memory_versions.list` / `retrieve` / `redact` | `MemoryStores.MemoryVersions.List` / `Get` / `Redact` |

**需要注意的命名差异：**
- Agents 和 Session Threads **没有 delete** — 只有 `archive`。Archive 是**永久性的**：Agent 变为只读，新会话无法引用它，且无法取消归档。在归档生产环境 Agent 之前，请先与用户确认。Environments、Sessions、Vaults、Credentials 和 Memory Stores 同时具有 `delete` 和 `archive`；Session Resources、Files、Skills 和 Memories 仅支持 `delete`；Memory Versions 两者都没有 — 只有 `redact`。
- Session resources 使用 `add`（而非 `create`）。
- Go 的事件流方法是 `StreamEvents`（而非 `Stream`）。
- 自托管 worker **不在** `client.beta.*` 下 — 它是 `anthropic.lib.environments` / `@anthropic-ai/sdk/helpers/beta/environments` 中的 `EnvironmentWorker`；只有 `environments.work.poller/stats/stop` 是客户端方法。

**Agent 简写形式：** 创建会话时，`agent` 接受三种形式 — 纯字符串（`agent="agent_abc123"`，使用最新版本）、固定引用 `{type: "agent", id, version}`，或 `{type: "agent_with_overrides", id, version?, model?, system?, tools?, mcp_servers?, skills?}` 以仅为此会话覆盖这些字段（参见 `shared/managed-agents-core.md` → 为会话覆盖 Agent 配置）。

**Model 简写形式：** 创建 Agent 时，`model` 接受纯字符串（`model="claude-opus-4-8"` — 使用 `standard` 速度）或完整配置对象（`{id: "claude-opus-4-8", speed: "fast"}`）。注意：`speed: "fast"` 仅在 Opus 4.8 和 Opus 4.7 上受支持。Opus 4.7 的快速模式已弃用；移除后，对 Opus 4.7 使用 `speed: "fast"` 将返回错误。Opus 4.8 是持久的快速能力层级。

---

## Agents

**每个流程的第一步。** 会话需要预先创建的 Agent — 在 `managed-agents-2026-04-01` 中没有内联 Agent 配置。

| 方法   | 路径                                             | 操作        | 描述                              |
| -------- | ------------------------------------------------ | ---------------- | ---------------------------------------- |
| `GET` | `/v1/agents` | ListAgents | 列出 Agents |
| `POST` | `/v1/agents` | CreateAgent | 创建已保存的 Agent 配置 |
| `GET` | `/v1/agents/{agent_id}` | GetAgent | 获取 Agent 详情 |
| `POST` | `/v1/agents/{agent_id}` | UpdateAgent | 更新 Agent 配置 |
| `POST` | `/v1/agents/{agent_id}/archive` | ArchiveAgent | 归档 Agent。使其变为**只读**；现有会话继续运行，新会话无法引用它。无法取消归档 — 这是终态。 |
| `GET` | `/v1/agents/{agent_id}/versions` | ListAgentVersions | 列出 Agent 版本 |

## Sessions

| 方法   | 路径                                             | 操作        | 描述                              |
| -------- | ------------------------------------------------ | ---------------- | ---------------------------------------- |
| `GET` | `/v1/sessions` | ListSessions | 列出会话（分页） |
| `POST` | `/v1/sessions` | CreateSession | 创建新会话 |
| `GET` | `/v1/sessions/{session_id}` | GetSession | 获取会话详情 |
| `POST` | `/v1/sessions/{session_id}` | UpdateSession | 更新会话的 `metadata`/`title`，或 `agent.tools`/`agent.mcp_servers`/`vault_ids`（会话本地覆盖；会话必须为 `idle`）。参见 `shared/managed-agents-core.md` → 在会话中途更新 Agent 配置。 |
| `DELETE` | `/v1/sessions/{session_id}` | DeleteSession | 删除会话 |
| `POST` | `/v1/sessions/{session_id}/archive` | ArchiveSession | 归档会话 |

## Events

| 方法   | 路径                                             | 操作        | 描述                              |
| -------- | ------------------------------------------------ | ---------------- | ---------------------------------------- |
| `GET` | `/v1/sessions/{session_id}/events` | ListEvents | 列出事件（轮询，分页） |
| `POST` | `/v1/sessions/{session_id}/events` | SendEvents | 发送事件（用户消息、工具结果） |
| `GET` | `/v1/sessions/{session_id}/events/stream` | StreamEvents | 通过 SSE 流式传输事件。可选的 `event_deltas[]=agent.message` / `agent.thinking` 参数可启用实时预览 `event_start`/`event_delta` 事件 — 参见 `shared/managed-agents-events.md` § 实时预览。 |

## Session Threads

多 Agent 会话中每个子 Agent 的事件流。参见 `shared/managed-agents-multiagent.md`。

| 方法   | 路径                                             | 操作        | 描述                              |
| -------- | ------------------------------------------------ | ---------------- | ---------------------------------------- |
| `GET` | `/v1/sessions/{session_id}/threads` | ListThreads | 列出 Threads（分页） |
| `GET` | `/v1/sessions/{session_id}/threads/{thread_id}` | GetThread | 获取单个 Thread（携带 `agent` 快照、`status`、`parent_thread_id`、`stats`、`usage`） |
| `POST` | `/v1/sessions/{session_id}/threads/{thread_id}/archive` | ArchiveThread | 归档 Thread |
| `GET` | `/v1/sessions/{session_id}/threads/{thread_id}/events` | ListThreadEvents | 列出单个 Thread 的历史事件（分页） |
| `GET` | `/v1/sessions/{session_id}/threads/{thread_id}/stream` | StreamThreadEvents | 通过 SSE 流式传输单个 Thread（SDK: `threads.events.stream`） |

## Session Resources

| 方法   | 路径                                                    | 操作        | 描述                              |
| -------- | ------------------------------------------------------- | ---------------- | ---------------------------------------- |
| `GET` | `/v1/sessions/{session_id}/resources` | ListResources | 列出附加到会话的资源 |
| `POST` | `/v1/sessions/{session_id}/resources` | AddResource | 附加 `file` 或 `github_repository` 资源（SDK 方法: `add`，而非 `create`）。`memory_store` 资源仅在创建会话时附加。 |
| `GET` | `/v1/sessions/{session_id}/resources/{resource_id}` | GetResource | 获取单个资源 |
| `POST` | `/v1/sessions/{session_id}/resources/{resource_id}` | UpdateResource | 更新资源 |
| `DELETE` | `/v1/sessions/{session_id}/resources/{resource_id}` | DeleteResource | 从会话中移除资源 |

## Environments

| 方法   | 路径                                                             | 操作            | 描述                         |
| -------- | ---------------------------------------------------------------- | -------------------- | ----------------------------------- |
| `POST`   | `/v1/environments`                                     | CreateEnvironment    | 创建环境                  |
| `GET`    | `/v1/environments`                                     | ListEnvironments     | 列出环境                   |
| `GET`    | `/v1/environments/{environment_id}`                    | GetEnvironment       | 获取环境详情             |
| `POST`   | `/v1/environments/{environment_id}`                    | UpdateEnvironment    | 更新环境                  |
| `DELETE` | `/v1/environments/{environment_id}`                    | DeleteEnvironment    | 删除环境。返回 204。 |
| `POST`   | `/v1/environments/{environment_id}/archive`            | ArchiveEnvironment   | 归档环境。使其变为**只读**；现有会话继续运行，新会话无法引用它。无法取消归档 — 这是终态。 |
| `GET`    | `/v1/environments/{environment_id}/work/stats`         | WorkQueueStats       | 自托管工作队列深度/待处理/worker 数量。`x-api-key` 认证。参见 `shared/managed-agents-self-hosted-sandboxes.md`。 |
| `POST`   | `/v1/environments/{environment_id}/work/{work_id}/stop` | StopWork            | 自托管：停止已认领的工作项。`x-api-key` 认证。 |

对于 `type: "self_hosted"`，`config` 是简单的 `{"type": "self_hosted"}` — `networking` 和 `packages` 不适用。

## Deployments

定时部署（`depl_` ID）按定期 cron 计划运行 Agent — 每次触发都会创建一个会话。参见 `shared/managed-agents-scheduled-deployments.md` 了解概念指南（cron/DST 语义、失败行为、生命周期）。

| 方法   | 路径                                             | 操作        | 描述                              |
| -------- | ------------------------------------------------ | ---------------- | ---------------------------------------- |
| `POST`   | `/v1/deployments`                                | CreateDeployment | 创建定时部署            |
| `POST`   | `/v1/deployments/{deployment_id}/pause`          | PauseDeployment  | 暂停定时触发（可逆；手动运行仍允许） |
| `POST`   | `/v1/deployments/{deployment_id}/unpause`        | UnpauseDeployment | 从下一个时间点恢复（不补执行） |
| `POST`   | `/v1/deployments/{deployment_id}/archive`        | ArchiveDeployment | **终态** — 计划停止，部署变为不可变 |
| `POST`   | `/v1/deployments/{deployment_id}/run`            | RunDeployment    | 立即触发手动运行（`trigger_context.type: "manual"`）；暂停时也可使用 |

## Deployment Runs

每次触发尝试（定时或手动）都会写入一条 `deployment_run` 记录（`drun_` ID），包含创建的 `session_id` 或 `error.type`（`environment_archived`、`agent_archived`、`vault_not_found`、`session_rate_limited`、`service_unavailable`）。

| 方法   | 路径                                             | 操作        | 描述                              |
| -------- | ------------------------------------------------ | ---------------- | ---------------------------------------- |
| `GET`    | `/v1/deployment_runs?deployment_id=...`          | ListDeploymentRuns | 列出部署的运行记录（分页；使用 `has_error=true` 过滤失败记录） |
| `GET`    | `/v1/deployment_runs/{deployment_run_id}`        | GetDeploymentRun   | 按 ID 获取单个运行记录（`deployment_run.*` webhook 事件中以此作为 `data.id`） |

## Vaults

Vaults 存储由 Anthropic 代为管理的凭证 — MCP 凭证（带自动刷新的 OAuth，或静态 bearer token）以及在出口处替换到外发请求中的 `environment_variable` 凭证。通过 `vault_ids` 附加到会话。参见 `managed-agents-tools.md` §Vaults 了解概念指南和凭证格式。

| 方法   | 路径                                             | 操作        | 描述                              |
| -------- | ------------------------------------------------ | ---------------- | ---------------------------------------- |
| `POST`   | `/v1/vaults`                                     | CreateVault      | 创建 vault                           |
| `GET`    | `/v1/vaults`                                     | ListVaults       | 列出 vaults                              |
| `GET`    | `/v1/vaults/{vault_id}`                          | GetVault         | 获取 vault 详情                        |
| `POST`   | `/v1/vaults/{vault_id}`                          | UpdateVault      | 更新 vault                             |
| `DELETE` | `/v1/vaults/{vault_id}`                          | DeleteVault      | 删除 vault                             |
| `POST`   | `/v1/vaults/{vault_id}/archive`                  | ArchiveVault     | 归档 vault                            |

## Credentials

Credentials 是存储在 vault 中的单独密钥。

| 方法   | 路径                                                              | 操作          | 描述                  |
| -------- | ----------------------------------------------------------------- | ------------------ | ---------------------------- |
| `POST`   | `/v1/vaults/{vault_id}/credentials`                               | CreateCredential   | 创建 credential          |
| `GET`    | `/v1/vaults/{vault_id}/credentials`                               | ListCredentials    | 列出 vault 中的 credentials    |
| `GET`    | `/v1/vaults/{vault_id}/credentials/{credential_id}`               | GetCredential      | 获取 credential 元数据      |
| `POST`   | `/v1/vaults/{vault_id}/credentials/{credential_id}`               | UpdateCredential   | 更新 credential            |
| `DELETE` | `/v1/vaults/{vault_id}/credentials/{credential_id}`               | DeleteCredential   | 删除 credential            |
| `POST`   | `/v1/vaults/{vault_id}/credentials/{credential_id}/archive`       | ArchiveCredential  | 归档 credential           |
| `POST`   | `/v1/vaults/{vault_id}/credentials/{credential_id}/mcp_oauth_validate` | McpOauthValidate | 验证 MCP OAuth credential |

## Memory Stores

工作区级别的持久化记忆，跨会话存活。通过 `resources[]` 中的 `{"type": "memory_store", "memory_store_id": ...}` 条目附加到会话（仅在创建会话时）。参见 `shared/managed-agents-memory.md` 了解概念指南、FUSE 挂载的 Agent 接口、前置条件和版本控制。

| 方法   | 路径                                             | 操作          | 描述                              |
| -------- | ------------------------------------------------ | ------------------ | ---------------------------------------- |
| `POST`   | `/v1/memory_stores`                              | CreateMemoryStore  | 创建存储（`name`、`description`、`metadata`） |
| `GET`    | `/v1/memory_stores`                              | ListMemoryStores   | 列出存储（`include_archived`、`created_at_{gte,lte}`） |
| `GET`    | `/v1/memory_stores/{memory_store_id}`            | GetMemoryStore     | 获取存储详情                        |
| `POST`   | `/v1/memory_stores/{memory_store_id}`            | UpdateMemoryStore  | 更新存储                             |
| `DELETE` | `/v1/memory_stores/{memory_store_id}`            | DeleteMemoryStore  | 删除存储                             |
| `POST`   | `/v1/memory_stores/{memory_store_id}/archive`    | ArchiveMemoryStore | 归档存储。使其变为**只读**；现有会话继续运行，新会话无法引用它。无法取消归档。 |

## Memories

存储中的单个文本文档（每个 ≤ 100KB）。`create` 在指定 `path` 处创建，如果路径已被占用则返回 `409`（`memory_path_conflict_error`，附带 `conflicting_memory_id`）；`update` 通过 `mem_...` ID 修改（重命名和/或内容）。只有 `update` 接受 `precondition`（`{"type": "content_sha256", "content_sha256": ...}`）— 不匹配时返回 `409`（`memory_precondition_failed_error`）。列表端点接受 `view: "basic"|"full"`（控制是否填充 `content`；`retrieve` 默认为 `full`）。

| 方法   | 路径                                                              | 操作      | 描述                              |
| -------- | ----------------------------------------------------------------- | -------------- | ---------------------------------------- |
| `GET`    | `/v1/memory_stores/{memory_store_id}/memories`                    | ListMemories   | 返回 `Memory \| MemoryPrefix`；按 `path_prefix`、`depth`、`order_by`/`order` 过滤 |
| `POST`   | `/v1/memory_stores/{memory_store_id}/memories`                    | CreateMemory   | 在 `path` 处创建（SDK: `memories.create`）；如果路径已占用则返回 `409 memory_path_conflict_error` |
| `GET`    | `/v1/memory_stores/{memory_store_id}/memories/{memory_id}`        | GetMemory      | 读取单条记忆（默认 `view="full"`） |
| `PATCH`  | `/v1/memory_stores/{memory_store_id}/memories/{memory_id}`        | UpdateMemory   | 按 ID 更改 `content`、`path` 或两者；可选 `precondition` |
| `DELETE` | `/v1/memory_stores/{memory_store_id}/memories/{memory_id}`        | DeleteMemory   | 删除（可选 `expected_content_sha256`） |

## Memory Versions

不可变的逐次变更快照（`memver_...`）— 审计和回滚的依据。`operation` ∈ `created` / `modified` / `deleted`。

| 方法   | 路径                                                                          | 操作             | 描述                              |
| -------- | ----------------------------------------------------------------------------- | --------------------- | ---------------------------------------- |
| `GET`    | `/v1/memory_stores/{memory_store_id}/memory_versions`                         | ListMemoryVersions    | 按最新优先排序；按 `memory_id`、`operation`、`session_id`、`api_key_id`、`created_at_{gte,lte}` 过滤 |
| `GET`    | `/v1/memory_stores/{memory_store_id}/memory_versions/{version_id}`            | GetMemoryVersion      | 列出字段 + 完整 `content`             |
| `POST`   | `/v1/memory_stores/{memory_store_id}/memory_versions/{version_id}/redact`     | RedactMemoryVersion   | 清除 `content`/`content_sha256`/`content_size_bytes`/`path`；保留操作者和时间戳 |

## Files

| 方法   | 路径                                             | 操作        | 描述                              |
| -------- | ------------------------------------------------ | ---------------- | ---------------------------------------- |
| `POST`   | `/v1/files`                            | UploadFile       | 上传文件                            |
| `GET`    | `/v1/files`                            | ListFiles        | 列出文件                               |
| `GET`    | `/v1/files/{file_id}`                  | GetFile          | 获取文件元数据（SDK 方法: `retrieve_metadata`） |
| `GET`    | `/v1/files/{file_id}/content`          | DownloadFile     | 下载文件内容                    |
| `DELETE` | `/v1/files/{file_id}`                  | DeleteFile       | 删除文件                            |

## Skills

| 方法   | 路径                                                            | 操作          | 描述                  |
| -------- | --------------------------------------------------------------- | ------------------ | ---------------------------- |
| `POST`   | `/v1/skills`                                          | CreateSkill        | 创建 skill               |
| `GET`    | `/v1/skills`                                          | ListSkills         | 列出 skills                  |
| `GET`    | `/v1/skills/{skill_id}`                               | GetSkill           | 获取 skill 详情            |
| `DELETE` | `/v1/skills/{skill_id}`                               | DeleteSkill        | 删除 skill               |
| `POST`   | `/v1/skills/{skill_id}/versions`                      | CreateVersion      | 创建 skill 版本         |
| `GET`    | `/v1/skills/{skill_id}/versions`                      | ListVersions       | 列出 skill 版本          |
| `GET`    | `/v1/skills/{skill_id}/versions/{version}`            | GetVersion         | 获取 skill 版本            |
| `DELETE` | `/v1/skills/{skill_id}/versions/{version}`            | DeleteVersion      | 删除 skill 版本         |

---

## 请求/响应 Schema 快速参考

### CreateAgent 请求体

**始终从这里开始。** `model`、`system`、`tools`、`mcp_servers`、`skills` 是此对象的顶层字段 — 它们不属于会话。

```json
{
  "name": "string (必填，1-256 字符)",
  "model": "claude-opus-4-8 (必填 — 纯字符串，或 {id, speed} 对象)",
  "description": "string (可选，最多 2048 字符)",
  "system": "string (可选，最多 100,000 字符)",
  "tools": [
    { "type": "agent_toolset_20260401" }
  ],
  "skills": [
    { "type": "anthropic", "skill_id": "xlsx" },
    { "type": "custom", "skill_id": "skill_abc123", "version": "1" }
  ],
  "mcp_servers": [
    {
      "type": "url",
      "name": "github",
      "url": "https://api.githubcopilot.com/mcp/"
    }
  ],
  "multiagent": {
    "type": "coordinator",
    "agents": [
      "agent_abc123",
      { "type": "agent", "id": "agent_def456", "version": 4 },
      { "type": "self" }
    ]
  },
  "metadata": {
    "key": "value (最多 16 对，键 ≤64 字符，值 ≤512 字符)"
  }
}
```

> 限制：`tools` 最多 128 个，`skills` 最多 20 个，`mcp_servers` 最多 20 个（名称唯一）。`multiagent.agents` 1–20 个条目（字符串 ID | `{type:"agent",id,version?}` | `{type:"self"}`）— 参见 `shared/managed-agents-multiagent.md`。

### CreateSession 请求体

```json
{
  "agent": "agent_abc123 (必填 — 最新版本的字符串简写，或 {type: \"agent\", id, version} 对象)",
  "environment_id": "env_abc123 (必填)",
  "title": "string (可选)",
  "resources": [
    {
      "type": "github_repository",
      "url": "https://github.com/owner/repo (必填)",
      "authorization_token": "ghp_... (必填)",
      "mount_path": "/workspace/repo (可选 — 默认为 /workspace/<repo-name>)",
      "checkout": { "type": "branch", "name": "main" }
    }
  ],
  "vault_ids": ["vlt_abc123 (可选 — vault 凭证：MCP 认证 + 环境变量)"],
  "metadata": {
    "key": "value"
  }
}
```

> `agent` 字段接受字符串 ID、`{type: "agent", id, version}`，或 `{type: "agent_with_overrides", id, version?, ...}` 用于会话本地覆盖 `model`/`system`/`tools`/`mcp_servers`/`skills`。除覆盖形式外，这些字段属于 Agent，不属于会话。
>
> **`checkout`** 接受 `{type: "branch", name: "..."}` 或 `{type: "commit", sha: "..."}`。省略则使用仓库的默认分支。

### CreateEnvironment 请求体

```json
{
  "name": "string (必填)",
  "description": "string (可选)",
  "config": {
    "type": "cloud | self_hosted",
    "networking": {
      "type": "unrestricted | limited (联合类型 — 参见 SDK 类型定义)"
    },
    "packages": { }
  },
  "metadata": { "key": "value" }
}
```

### CreateDeployment 请求体

```json
{
  "name": "Weekly compliance scan",
  "agent": "agent_abc123 (必填 — 与 CreateSession 相同的格式)",
  "environment_id": "env_abc123 (必填)",
  "initial_events": [
    { "type": "user.message", "content": [{ "type": "text", "text": "Run the weekly compliance scan." }] }
  ],
  "schedule": {
    "type": "cron",
    "expression": "0 20 * * 5",
    "timezone": "America/New_York"
  }
}
```

> 可选的会话配置（`resources`、`vault_ids` 等）的支持方式与 CreateSession 相同。响应包含 `status`、`paused_reason` 和 `schedule.upcoming_runs_at`（下次触发时间）。参见 `shared/managed-agents-scheduled-deployments.md`。

### SendEvents 请求体

```json
{
  "events": [
    {
      "type": "user.message",
      "content": [
        {
          "type": "text",
          "text": "Hello"
        }
      ]
    }
  ]
}
```

> `system.message` 事件（在轮次之间更新系统提示词）使用相同的信封格式，`type: "system.message"` — 仅限 Claude Opus 4.8；参见 `shared/managed-agents-events.md` § 在会话中途更新系统提示词。

### Define Outcome 事件

```json
{
  "type": "user.define_outcome",
  "description": "Build a DCF model for Costco in .xlsx",
  "rubric": { "type": "file", "file_id": "file_01..." },
  "max_iterations": 5
}
```

> `rubric` 为必填：`{type: "text", content}` 或 `{type: "file", file_id}`。`max_iterations` 默认 3，最大 20。响应会回显 `outcome_id` + `processed_at`。参见 `shared/managed-agents-outcomes.md`。

### 工具结果事件

```json
{
  "type": "user.custom_tool_result",
  "custom_tool_use_id": "sevt_abc123",
  "content": [{ "type": "text", "text": "Result data" }],
  "is_error": false
}
```

---

## 错误处理

Managed Agents 端点使用标准的 Anthropic API 错误格式。错误以 HTTP 状态码和包含 `type`、`error`、`request_id` 的 JSON 响应体返回：

```json
{
  "type": "error",
  "error": {
    "type": "invalid_request_error",
    "message": "Description of what went wrong"
  },
  "request_id": "req_011CRv1W3XQ8XpFikNYG7RnE"
}
```

向 Anthropic 报告问题时请包含 `request_id` — 它使我们能够端到端追踪请求。内部的 `error.type` 为以下之一：

| 状态码 | 错误类型 | 描述 |
|---|---|---|
| 400 | `invalid_request_error` | 请求格式错误或缺少必填参数 |
| 401 | `authentication_error` | API 密钥无效或缺失 |
| 403 | `permission_error` | API 密钥没有此操作的权限 |
| 404 | `not_found_error` | 请求的资源不存在 |
| 409 | `invalid_request_error` | 请求与资源的当前状态冲突（例如，向已归档的会话发送消息） |
| 413 | `request_too_large` | 请求体超过最大允许大小 |
| 429 | `rate_limit_error` | 请求过多 — 检查速率限制请求头以获取重试时间 |
| 500 | `api_error` | 内部服务器错误 |
| 529 | `overloaded_error` | 服务暂时过载 — 使用退避策略重试 |

注意 `409 Conflict` 携带 `error.type: "invalid_request_error"`（没有单独的 `conflict_error` 类型）；需要同时检查 HTTP 状态码和 `message` 来区分冲突和其他无效请求。

---

## 分页

大多数 Managed Agents 列表端点使用 `page` / `next_page` 游标方案：

| 字段 | 位置 | 说明 |
|---|---|---|
| `limit` | 查询参数 | 每页最大条目数 |
| `page` | 查询参数 | 来自上一个响应的不透明游标 — 将 `next_page` 或 `prev_page` 值传入此处 |
| `order` | 查询参数 | 在支持排序的端点上为 `asc` / `desc`。游标编码了生成它的请求的 `order` — 以不同的 `order` 重用同一游标会返回 400。其他参数（过滤器、`limit`）可以在分页请求之间更改。 |
| `next_page` | 响应 | 下一页的游标；没有更多结果时为 `null` |
| `prev_page` | 响应 | 支持向后分页的端点上的上一页游标 — 目前**仅 `GET /v1/sessions`**。第一页时为 `null`。在不支持此功能的端点上，该字段**不存在**（不是 `null`）。 |

每个 SDK 都提供自动分页迭代器，自动跟踪 `next_page`。在 Python 和 TypeScript 中，直接迭代列表结果；其他 SDK 通过单独的方法暴露迭代器（迭代普通列表结果只返回一页）。SDK 自动分页是**单向的** — 要返回上一页，需从响应中读取 `prev_page` 并将其作为 `page` 参数传回。

> ⚠️ 部分端点使用**不同的**游标方案：Message Batches、Files、Models 以及部分 Admin API 端点接受 `after_id`/`before_id` 并返回 `has_more`/`first_id`/`last_id`，而非 `page`/`next_page`。部分 `page` 方案的端点（如 `GET /v1/skills`）还会在 `next_page` 旁边返回 `has_more` 布尔值。请查看端点的参考页面了解其具体的分页字段。

---

## 速率限制

Managed Agents 端点有按组织的每分钟请求数（RPM）限制，与 [Messages API 的 token 限制](https://platform.claude.com/docs/en/api/rate-limits)分开计算。会话内的模型推理仍然消耗组织的标准 ITPM/OTPM 限制。

| 端点组 | 范围 | RPM | 最大并发数 |
|---|---|---|---|
| 创建操作（Agents、Sessions、Vaults） | 组织 | 300 | — |
| 所有其他操作（Agents、Sessions、Vaults） | 组织 | 600 | — |
| 所有操作（Environments） | 组织 | 60 | 5 |

Files 和 Skills 端点使用基于等级的标准[速率限制](https://platform.claude.com/docs/en/api/rate-limits)。

当超出限制时，API 返回 `429` 和 `rate_limit_error`（响应信封参见[错误处理](#错误处理)），以及 `retry-after` 请求头指示等待多少秒后重试。Anthropic SDK 会读取此请求头并自动重试。