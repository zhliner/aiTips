# Managed Agents — 核心概念

## 架构

Managed Agents 围绕四个核心概念构建：

| 概念 | 端点 | 说明 |
|---|---|---|
| **Agent** | `/v1/agents` | 一个持久化的、带版本的对象，定义 Agent 的能力和角色：模型、系统提示词、工具、MCP 服务器、Skills。**必须在启动会话之前创建。** 参见下方 Agents 部分。 |
| **Session** | `/v1/sessions` | 与 Agent 的有状态交互。通过 ID 引用预先创建的 Agent + 环境 + 初始指令。产生事件流。 |
| **Environment** | `/v1/environments` | 定义容器配置模板。 |
| **Container** | N/A | 一个隔离的计算实例，Agent 的**工具**在此执行（bash、文件操作、代码）。Agent 循环不在此运行 — 它在 Anthropic 的编排层运行，并通过工具调用操作容器。 |

```
                       ┌─────────────────────────────────────┐
                       │  Anthropic 编排层                    │
Agent (配置) ──────────▶│  (Agent 循环：Claude + 工具调用)      │
                       └──────────────┬──────────────────────┘
                                      │ 工具调用
                                      ▼
Environment (模板) ──▶ Container (工具执行工作区)
                                 │
                         Session ─┤
                                 ├── Resources (文件、仓库、Memory Stores — 启动时附加)
                                 ├── Vault IDs (MCP 凭证引用)
                                 └── Conversation (事件流输入/输出)
```

> **Agent 创建是前提条件。** 会话通过 ID 引用预先创建的 Agent — `model`/`system`/`tools` 属于 Agent 对象，不属于会话。每个流程都从 `POST /v1/agents` 开始。

---

## 会话生命周期

```
rescheduling → running ↔ idle → terminated
```

| 状态         | 描述                                                        |
| -------------- | ------------------------------------------------------------------ |
| `idle` | Agent 已完成当前任务，正在等待输入。它在等待通过 `user.message` 继续工作的输入，或者被阻塞等待 `user.custom_tool_result` 或 `user.tool_confirmation`。附加的 `stop_reason` 包含关于 Agent 停止工作的更多信息。 |
| `running` | 会话已开始运行，Agent 正在积极工作。 |
| `rescheduling` | 会话在发生可重试错误后正在（重新）调度，等待被编排系统接管。 |
| `terminated` | 会话已终止，进入不可逆且不可用的状态。  |

- 事件可以在会话处于 `running` 或 `idle` 时发送。消息按顺序排队处理。
- Agent 在收到新事件时从 `idle → running` 转换，完成后回到 `idle`。
- 错误以流中的 `session.error` 事件形式呈现，而不是作为状态值。

每个会话在 Anthropic Console 中都有实时追踪视图，地址为 `https://platform.claude.com/workspaces/default/sessions/{session_id}`。在创建会话后立即打印此 URL，让用户可以实时观看工具调用和消息流。`default` 工作区段会在加载时自动解析为会话的实际工作区，因此你不需要工作区 ID。

### 内置会话功能

- **上下文压缩** — 如果接近最大上下文，API 会自动压缩会话历史以保持交互继续
- **提示词缓存** — 历史重复 token 会被缓存，减少处理时间和成本
- **扩展思考** — 默认开启，以 `agent.thinking` 事件形式返回

### 会话操作

| 操作 | 说明 |
|---|---|
| 列出 / 获取 | 分页列表或按 ID 获取单个资源 |
| 更新 | 仅 `title` 可更新 |
| 归档 | 会话变为**只读**。不可逆。 |
| 删除 | 永久删除会话、事件历史、容器和检查点。 |

这些是运维/检查调用 — 通常从终端发起，而非应用代码。从命令行（参见 `shared/anthropic-cli.md`）：

```sh
ant beta:sessions list --transform '{id,title,status,created_at}' --format jsonl
ant beta:sessions retrieve --session-id "$SID"
ant beta:sessions:events stream --session-id "$SID"   # 实时观看事件
ant beta:sessions archive  --session-id "$SID"
ant beta:sessions delete   --session-id "$SID"
```

---

## Sessions

会话是在环境中运行的 Agent 实例。

### 会话对象

API 返回的关键字段：

| 字段           | 类型     | 描述                                         |
| --------------- | -------- | --------------------------------------------------- |
| `type` | string | 始终为 `"session"` |
| `id` | string | 唯一的会话 ID |
| `title` | string | 人类可读的标题 |
| `status` | string | `idle`、`running`、`rescheduling`、`terminated` |
| `created_at` | string | ISO 8601 时间戳 |
| `updated_at` | string | ISO 8601 时间戳 |
| `archived_at` | string | ISO 8601 时间戳（可为 null） |
| `environment_id` | string | 环境 ID |
| `agent` | object | Agent 配置 |
| `resources` | array | 附加的文件、仓库和 Memory Stores |
| `metadata` | object | 用户提供的键值对（最多 8 个键） |
| `usage` | object | Token 使用统计 |

### 创建会话

**没有 Agent，会话就没有意义。** 会话通过 ID 引用预先创建的 Agent。先通过 `agents.create()` 创建 Agent，然后引用它：

```ts
// 1. 创建 Agent（可复用，带版本）
const agent = await client.beta.agents.create(
  {
    name: "Coding Assistant",
    model: "claude-opus-4-8",
    system: "You are a helpful coding agent.",
    tools: [{ type: "agent_toolset_20260401"}],
  },
);

// 2. 启动引用该 Agent 的会话
const session = await client.beta.sessions.create(
  {
    agent: agent.id,  // 字符串简写 → 最新版本。或：{ type: "agent", id: agent.id, version: agent.version }
    environment_id: environmentId,
    title: "Hello World Session",
  },
);
```

> 💡 **在 Console 中实时观看。** 开发时，打印一个链接以便点击跳转到会话 UI，而不是解析事件流：`https://platform.claude.com/workspaces/default/sessions/{session.id}`。`default` 工作区段会在加载时自动解析为会话的实际工作区，因此你不需要知道工作区 ID。在你为用户生成的本地迭代代码中包含此 `print`/`console.log`。

**会话创建参数：**

| 字段           | 类型     | 必填 | 描述                                    |
| --------------- | -------- | -------- | ---------------------------------------------- |
| `agent`         | string 或 object | **是** | 三种形式：字符串简写 `"agent_abc123"`（最新版本）；固定引用 `{type: "agent", id, version}`；或 `{type: "agent_with_overrides", id, version?, ...}` 仅为此会话覆盖 `model`/`system`/`tools`/`mcp_servers`/`skills` — 参见 § 为会话覆盖 Agent 配置 |
| `environment_id`| string   | **是**  | 环境 ID                                 |
| `title`         | string   | 否       | 人类可读的名称（出现在日志/仪表板中） |
| `resources`     | array    | 否       | 文件、GitHub 仓库或 Memory Stores，在启动时附加到容器。Memory Stores 仅在创建会话时附加（不能通过 `resources.add()` 添加）。 |
| `vault_ids`     | array    | 否       | Vault ID（`vlt_*`）— 带自动刷新的 MCP 凭证 + 在出口处替换的 `environment_variable` 密钥。参见 `shared/managed-agents-tools.md` → Vaults。 |
| `metadata`      | object   | 否       | 用户提供的键值对                  |

**Agent 配置字段**（传给 `agents.create()`，而非 `sessions.create()`）：

| 字段         | 类型     | 必填 | 描述                                    |
| ------------- | -------- | -------- | ---------------------------------------------- |
| `name`        | string   | **是**  | 人类可读的名称（1-256 字符）              |
| `model`       | string 或 object | **是** | Claude 模型 ID（纯字符串，或 `{id, speed}` 对象）。支持所有 Claude 4.5+ 模型。 |
| `system`      | string   | 否       | 系统提示词 — 定义 Agent 的行为（最多 100K 字符） |
| `tools`       | array    | 否       | 包含三种类型：(1) 预构建的 Claude Agent 工具（`agent_toolset_20260401`），(2) MCP 工具（`mcp_toolset`），(3) 自定义客户端工具。最多 128 个。 |
| `mcp_servers` | array    | 否       | MCP 服务器连接 — 标准化的第三方能力（如 GitHub、Asana）。最多 20 个，名称唯一。参见 `shared/managed-agents-tools.md` → MCP Servers。 |
| `skills`      | array    | 否       | 带渐进式披露的定制化"最佳实践"上下文。最多 20 个。参见 `shared/managed-agents-tools.md` → Skills。 |
| `description` | string   | 否       | Agent 的描述（最多 2048 字符）    |
| `multiagent`  | object   | 否       | `{type: "coordinator", agents: [...]}` — 此 Agent 可委托到的 Agent 名册。参见 `shared/managed-agents-multiagent.md`。 |
| `metadata`    | object   | 否       | 任意键值对（最多 16 个，键 ≤64 字符，值 ≤512 字符） |

---

## Agents

**这是每个 Managed Agents 流程的起点。** Agent 对象是一个持久化的、带版本的配置 — 你创建一次，然后每次启动会话时通过 ID 引用它。没有 Agent → 没有会话。

### Agent 对象

API 是**扁平的** — `model`、`system`、`tools` 等是顶层字段，不包裹在 `agent:{}` 子对象中。

| 字段              | 类型     | 必填 | 描述                                        |
| ------------------ | -------- | -------- | -------------------------------------------------- |
| `name`             | string   | 是      | 人类可读的名称                                |
| `model`            | string   | 是      | Claude 模型 ID                                    |
| `system`           | string   | 否       | 系统提示词                                      |
| `tools`            | array    | 否       | Agent 工具集 / MCP 工具集 / 自定义工具         |
| `mcp_servers`      | array    | 否       | MCP 服务器连接                             |
| `skills`           | array    | 否       | Skill 引用（最多 20 个）                          |
| `description`      | string   | 否       | Agent 的描述                           |
| `multiagent`       | object   | 否       | 协调者名册 — 参见 `shared/managed-agents-multiagent.md` |
| `metadata`         | object   | 否       | 任意键值对                          |

### 生命周期：创建一次，运行多次，就地更新

Agent 是**持久资源**，不是每次运行的参数。预期的模式：

```
┌─ 设置（一次性）─────────┐     ┌─ 运行时（每次调用）──────────┐
│ agents.create()        │     │ sessions.create(             │
│   → 存储 agent_id      │ ──→ │   agent={type:..., id: ID}   │
│     到配置/环境/数据库   │     │ )                            │
└────────────────────────┘     └──────────────────────────────┘
```

**反模式：** 在每次脚本运行时调用 `agents.create()`。这会积累孤立的 Agent 对象，每次调用都付出创建延迟，并且违背了版本模型。如果你在按请求或按 cron 周期调用的函数中看到 `agents.create()`，那就是错误的 — 将其提升到一次性设置中并持久化 ID。

> **推荐 — 将 Agents 和 Environments 定义为 YAML + 通过 `ant` CLI 应用。** 分工是 **CLI 用于控制面，SDK 用于数据面**：Agents 和 Environments 是相对静态的资源，你用 `ant` 管理它们（版本控制的 YAML，从 CI 应用）；Sessions 是动态的，由你的应用通过 SDK 驱动。参见 `shared/anthropic-cli.md` → *版本控制的 Managed Agents 资源* 了解 `ant beta:agents create < agent.yaml` / `update --version N` 流程。本文档中展示的 SDK `agents.create()` 调用是代码中的等价操作 — 在需要编程化配置时使用它，但对于由人类维护的内容，优先使用 YAML 流程。

### 版本控制

每次 `POST /v1/agents/{id}`（更新）都会创建一个新的不可变版本（数字时间戳，如 `1772585501101368014`）。Agent 的历史是仅追加的 — 你无法编辑过去的版本。

**为什么需要版本：**
- **可复现性** — 将会话固定到已知良好的配置：`{type: "agent", id, version: 3}`
- **安全迭代** — 更新 Agent 而不影响已在旧版本上运行的会话
- **回滚** — 如果新的系统提示词导致退化，将新会话固定回之前的版本，同时调试问题

**`version` 是可选的。** 省略它（或使用字符串简写 `agent="agent_abc123"`）以在创建会话时获取最新版本。显式传递（`{type: "agent", id, version: N}`）以固定用于可复现性。

**获取要固定的版本：** `agents.create()` 和 `agents.update()` 都在响应中返回 `version`。将其与 `agent_id` 一起存储。要获取现有 Agent 的当前最新版本：`GET /v1/agents/{id}` → `.version`。

**何时更新 vs 创建新的：** 当概念上是同一个 Agent 只是调整了行为（更好的提示词、额外的工具）时，使用更新（`POST /v1/agents/{id}`）。当是不同角色/用途时，创建新的 Agent。经验法则：如果你会给它相同的 `name`，就更新。

### Agent 端点

| 操作        | 方法   | 路径                                  |
| ---------------- | -------- | ------------------------------------- |
| 创建           | `POST`   | `/v1/agents`                          |
| 列出             | `GET`    | `/v1/agents`                          |
| 获取              | `GET`    | `/v1/agents/{id}`                     |
| 更新           | `POST`   | `/v1/agents/{id}`                     |
| 归档          | `POST`   | `/v1/agents/{id}/archive`             |

> ⚠️ **归档是永久性的。** 归档使 Agent 变为只读：现有会话继续运行，但**新会话无法引用它**，且无法取消归档。由于 Agents 没有 `delete`，这是终态生命周期状态。不要将归档生产环境 Agent 作为常规清理 — 请先与用户确认。

### 在会话中使用 Agent

通过字符串 ID（最新版本）或带显式版本的对象引用 Agent：

```python
# 字符串简写 — 使用 Agent 的最新版本
session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=environment_id,
)

# 或固定到特定版本（int）
session = client.beta.sessions.create(
    agent={"type": "agent", "id": agent.id, "version": agent.version},
    environment_id=environment_id,
)
```

### 为会话覆盖 Agent 配置

第三种 `agent` 形式 `agent_with_overrides`，为**单个会话**替换 Agent 配置的部分内容 — 尝试不同的模型或授予额外的工具，而无需对 Agent 进行版本化。传递 `id`（以及可选的 `version`；省略 = 最新版本，与其他两种形式的默认值相同）加上 `model`、`system`、`tools`、`mcp_servers`、`skills` 中的任意字段：

```python
session = client.beta.sessions.create(
    agent={
        "type": "agent_with_overrides",
        "id": agent.id,
        "model": "claude-opus-4-8",   # 为此会话替换 Agent 的模型
        "system": None,           # 为此会话清除系统提示词
    },
    environment_id=environment_id,
)
```

每个可覆盖字段遵循三态规则：
- **省略** → 会话从引用的 Agent 版本继承该值。
- **`null`（或列表字段的 `[]`）** → 会话在清除该字段的情况下运行。完全适用于 `system`、`mcp_servers`、`skills`。两个例外：`model` 不可清除（`model: null` → 400 `agent_model_required`）；当会话有效的 `skills` 非空时，清除 `tools` 返回 400（Skills 需要 `read` 工具），否则 `tools: null` / `tools: []` 可清除。
- **一个值** → **完全**替换 Agent 的值。覆盖从不合并 — `tools` 覆盖必须列出会话应拥有的所有工具。

覆盖是会话本地的：它们**不会**修改 Agent 资源或创建新的 Agent 版本。响应中的 `agent` 对象反映覆盖后的配置，而其 `id` 和 `version` 仍然标识基础 Agent — 因此你可以将会话追溯到其基础。在多 Agent 会话中，覆盖适用于协调者及其 `{type: "self"}` 副本；通过 ID 引用的名册中的 Agent 始终使用其创建时的配置（参见 `shared/managed-agents-multiagent.md`）。

### 在会话中途更新 Agent 配置

`sessions.update()` 可以更改**现有**会话的 `agent.tools`、`agent.mcp_servers`（包括权限策略）和 `vault_ids`。这是**会话本地覆盖** — 它不会创建新的 Agent 版本，也不会传播回 Agent 对象。提供的数组是**完全替换**；要追加一个工具，先 `GET` 会话，修改后 `POST` 回去。会话必须处于 `idle` 状态 — 如果正在运行，先中断。

只有 `tools` 和 `mcp_servers` 可以在会话创建后更改 — 要使用与 Agent 值不同的 `model`、`system` 或 `skills` 运行，请在创建时使用 `agent_with_overrides`（如上）。Agent 配置的 `system` 字段在会话生命周期内是固定的；你仍然可以通过发送 `system.message` 事件**在轮次之间替换有效的系统提示词**（参见 `shared/managed-agents-events.md` § 在会话中途更新系统提示词）。

```python
client.beta.sessions.update(
    session.id,
    agent={
        "tools": [
            {"type": "agent_toolset_20260401"},
            {"type": "mcp_toolset", "mcp_server_name": "linear"},
        ],
        "mcp_servers": [{"type": "url", "name": "linear", "url": "https://mcp.linear.app/sse"}],
    },
    vault_ids=["vlt_..."],
)
```