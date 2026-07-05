# Managed Agents — 概述

Managed Agents 为每个会话配置一个容器作为 Agent 的工作区。Agent 循环运行在 Anthropic 的编排层上；容器是 Agent 的*工具*执行的地方——bash 命令、文件操作、代码。你创建一个持久化的 **Agent** 配置（模型、系统提示、工具、MCP 服务器、技能），然后启动引用它的**会话**。会话将事件流式传输回你；你发送用户消息和工具结果。

## ⚠️ 必须遵循的流程：Agent（一次）→ 会话（每次运行）

**为什么 Agent 是独立对象：版本控制。** Agent 是一个持久化的、可版本化的配置——每次更新创建一个新的不可变版本，会话在创建时固定到某个版本。这使你可以在不中断已运行会话的情况下迭代 Agent（调整提示、添加工具），在变更导致回退时回滚，并并行 A/B 测试不同版本。如果你每次运行都 `agents.create()` 创建新的，以上功能都无法实现。

每个会话引用一个预先创建的 `/v1/agents` 对象。创建 Agent 一次，存储 ID，并在多次运行中复用。

| 步骤 | 调用 | 频率 |
|---|---|---|
| 1 | `POST /v1/agents` — `model`、`system`、`tools`、`mcp_servers`、`skills` 在此定义 | **一次。** 保存 `agent.id` **和** `agent.version`。 |
| 2 | `POST /v1/sessions` — `agent: "agent_abc123"` 或 `{type: "agent", id, version}` | **每次运行。** 字符串简写使用最新版本。 |

如果你正要编写带有 `model`、`system` 或 `tools` 的 `sessions.create()`——**请停下**。这些字段在 `agents.create()` 上。会话只接受*指针*。

**生成代码时，将设置与运行时分离。** `agents.create()` 属于设置脚本（或受保护的 `if agent_id is None:` 代码块），而非热路径的顶部。如果用户的代码在每次调用时都执行 `agents.create()`，他们会在累积孤立 Agent 的同时白白付出创建延迟。正确的模式是：创建一次 → 持久化 ID（配置文件、环境变量、密钥管理器）→ 每次运行加载 ID 并调用 `sessions.create()`。

**要更改 Agent 的行为，使用 `POST /v1/agents/{id}` ——不要创建新的。** 每次更新递增版本号；运行中的会话保持其固定的版本，新会话获取最新版本（或通过 `{type: "agent", id, version}` 显式固定）。参见 `shared/managed-agents-core.md` → Agent → 版本控制。要在**不修改 Agent 对象**的情况下更改**单个运行中会话**的 `tools`/`mcp_servers`/`vault_ids`，使用 `sessions.update()`——参见 `shared/managed-agents-core.md` → 在会话中途更新 Agent 配置。

## Beta Header

Managed Agents 处于 beta 阶段。SDK 自动设置所需的 beta header：

| Beta Header | 启用功能 |
| --- | --- |
| `managed-agents-2026-04-01` | Agent、Environment、Session、Event、Session Resource、Session Thread、Outcome、Multiagent、Vault、Credential、Memory Store、Deployment |
| `skills-2025-10-02` | Skills API（用于管理自定义技能定义） |
| `files-api-2025-04-14` | Files API，用于文件上传 |

**Beta header 的使用位置：** SDK 在 `client.beta.{agents,environments,sessions,vaults,memory_stores,deployments,deployment_runs}.*` 调用中自动设置 `managed-agents-2026-04-01`，在 `client.beta.files.*` / `client.beta.skills.*` 调用中自动设置 `files-api-2025-04-14` / `skills-2025-10-02`。调用 Managed Agents 端点时，你**不需要**手动添加 Skills 或 Files 的 beta header。**例外——会话级文件列表：** `client.beta.files.list({scope_id: session.id})` 是一个接受 Managed Agents 参数的 Files 端点，因此需要**两个** header。在该调用中显式传入 `betas: ["managed-agents-2026-04-01"]`（SDK 添加 Files header；你添加 Managed Agents header）。参见 `shared/managed-agents-environments.md` → 会话产出物。


## 阅读指南

| 用户想要... | 阅读这些文件 |
| --- | --- |
| **从零开始 / "帮我设置一个 Agent"** | `shared/managed-agents-onboarding.md` — 引导式访谈（WHERE→WHO→WHAT→WATCH），然后生成代码 |
| 了解 API 的工作方式 | `shared/managed-agents-core.md` |
| 查看完整的端点参考 | `shared/managed-agents-api-reference.md` |
| **创建 Agent**（必需的第一步） | `shared/managed-agents-core.md`（Agent 章节）+ 语言文件 |
| 更新/版本化 Agent | `shared/managed-agents-core.md`（Agent → 版本控制）——更新，而非重新创建 |
| 创建会话 | `shared/managed-agents-core.md` + `{lang}/managed-agents/README.md`（cURL/C#：`curl/managed-agents.md`） |
| 配置工具和权限 | `shared/managed-agents-tools.md` |
| 设置 MCP 服务器 | `shared/managed-agents-tools.md`（MCP 服务器章节） |
| 流式接收事件 / 处理 tool_use | `shared/managed-agents-events.md` + 语言文件 |
| 通过 webhook 获取会话状态变更通知（无需轮询） | `shared/managed-agents-webhooks.md` — Console 注册的端点、HMAC 验证、精简有效载荷 + 获取 |
| 定义 Outcome / 评分标准驱动的迭代循环 | `shared/managed-agents-outcomes.md` — `user.define_outcome` 事件、评分器、`span.outcome_evaluation_*` 事件 |
| 协调多个 Agent / 子 Agent / 线程 | `shared/managed-agents-multiagent.md` — Agent 上的 `multiagent: {type: "coordinator", agents: [...]}`、会话线程、跨线程工具确认 |
| 设置环境 | `shared/managed-agents-environments.md` + 语言文件 |
| 在你自己的基础设施 / VPC 中运行工具执行（自托管沙箱） | `shared/managed-agents-self-hosted-sandboxes.md` — `config:{type:"self_hosted"}`、`ANTHROPIC_ENVIRONMENT_KEY`、`EnvironmentWorker.run()` / `ant beta:worker poll` |
| 上传文件 / 附加仓库 | `shared/managed-agents-environments.md`（Resources） |
| 让 Agent 跨会话拥有持久记忆 | `shared/managed-agents-memory.md` — Memory Store、`memory_store` 会话资源、前置条件、版本/编辑 |
| 将 Agent/环境定义为版本控制的 YAML；从 shell 驱动 API | `shared/anthropic-cli.md` — `ant beta:agents create < agent.yaml`、`--transform`、`@file` 内联 |
| 存储凭据（MCP 认证、CLI/SDK 的 API 密钥） | `shared/managed-agents-tools.md`（Vault 章节）——`mcp_oauth` / `static_bearer` / `environment_variable` |
| 调用需要密钥的非 MCP API / CLI | `shared/managed-agents-tools.md`（Vault 章节）——`environment_variable` 凭据，在出站时替换。如果不适用（例如自托管沙箱），`shared/managed-agents-client-patterns.md` 模式 9 通过自定义工具将密钥保留在主机端 |
| 按定时计划运行 Agent | `shared/managed-agents-scheduled-deployments.md` — 部署、部署运行、暂停/自动暂停 |

## 常见陷阱

- **先创建 Agent，再创建会话——没有例外**——会话的 `agent` 字段仅接受字符串 ID 或 `{type: "agent", id, version}`。`model`、`system`、`tools`、`mcp_servers`、`skills` 是 **`POST /v1/agents` 上的顶级字段**，永远不在 `sessions.create()` 上。如果用户还没有创建 Agent，那是每个示例的第零步。
- **Agent 只创建一次，不是每次运行**——`agents.create()` 是设置步骤。保存返回的 `agent_id` 并复用；不要在热路径顶部调用 `agents.create()`。如果需要更改 Agent 配置，使用 `POST /v1/agents/{id}`——每次更新创建新版本，会话可以固定到特定版本以保证可复现性。
- **MCP 认证通过 Vault**——Agent 的 `mcp_servers` 数组仅声明 `{type, name, url}`（不含认证）。凭据存储在 Vault 中（`client.beta.vaults.credentials.create`）并通过 `vault_ids` 附加到会话。Anthropic 使用存储的 refresh token 自动刷新 OAuth token。Vault 还保存非 MCP 服务（CLI、SDK、直接 API 调用）的 `environment_variable` 凭据——在出站时替换，在沙箱中永远不可见。
- **在首次运行前协调资源**——一个有明确需求但缺少工具、凭据、数据挂载或上下文的会话会在运行中途发现缺口，然后陷入混乱并放弃。创建会话之前，检查任务中的每个操作是否映射到已配置的工具/MCP 服务器、每个 MCP 服务器是否有 Vault 凭据、每个引用的文件/主机是否已挂载/可达。帮助用户设置时，运行 `shared/managed-agents-onboarding.md` → §3 预检可行性检查中的协调流程。
- **使用流接收事件**——`GET /v1/sessions/{id}/events/stream` 是实时接收 Agent 输出的主要方式。
- **SSE 流没有重放——重连时使用合并**——如果流在 `agent.tool_use`、`agent.mcp_tool_use` 或 `agent.custom_tool_use` 等待解决时断开（前两者需要 `user.tool_confirmation`，最后一个需要 `user.custom_tool_result`），会话会死锁（客户端断开 → 会话空闲 → 重连发生 → 没有客户端解决）。每次（重）连接时：使用 `GET /v1/sessions/{id}/events/stream` 打开流，获取 `GET /v1/sessions/{id}/events`，按事件 ID 去重，然后继续。参见 `shared/managed-agents-events.md` → 流断开后的重连。
- **不要信任 HTTP 库的超时作为绝对时间上限**——`requests` 的 `timeout=(c, r)` 和 `httpx.Timeout(n)` 是*每块*读取超时；它们在每收到一个字节时重置，因此一个缓慢的连接可能无限阻塞。对于原始 HTTP 轮询的硬性截止时间，在循环级别跟踪 `time.monotonic()` 并显式退出。优先使用 SDK 的 `sessions.events.stream()` / `session.events.list()` 而非手写 HTTP。参见 `shared/managed-agents-events.md` → 接收事件。
- **消息会排队**——你可以在会话 `running` 或 `idle` 时发送事件；它们按顺序处理。无需等待响应即可发送下一条消息。
- **环境的 `config.type` 为 `"cloud"` 或 `"self_hosted"`**——`cloud` 在 Anthropic 的基础设施上运行容器；`self_hosted` 将工具执行转移到你自己的基础设施（参见 `shared/managed-agents-self-hosted-sandboxes.md`）。
- **归档对所有资源都是永久的**——归档 Agent、环境、会话、Vault、Credential 或 Memory Store 使其变为只读且不可撤销。特别是对于 Agent、环境和 Memory Store，已归档的资源不能被新会话引用（现有会话继续运行）。不要将 `.archive()` 作为清理手段调用到生产环境的 Agent、环境或 Memory Store 上——**归档前务必与用户确认**。
