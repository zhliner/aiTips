# Managed Agents — 入门流程

> **通过 `/claude-api managed-agents-onboard` 调用？** 你找对地方了。运行下面的访谈——不要将其总结后回给用户，直接提问。

Claude Managed Agents 是一个托管式 Agent：Anthropic 运行 Agent 循环，并为每个会话配置一个沙箱化容器来执行 Agent 的工具（或使用你自己的 worker，配合 `self_hosted` 环境——参见 `shared/managed-agents-self-hosted-sandboxes.md`）。你提供 **Agent 配置**（工具、技能、模型、系统提示——可复用、可版本化）和**环境配置**（沙箱——可跨 Agent 复用）。每次运行是一个**会话**。

流程分为四个步骤——**描述 → Agent → 环境 → 会话**——与 Console 快速入门的弧线相同，理念也相同：**先体现价值，再要求凭据**。用户在任何认证请求之前就能从想法到可运行的会话；每个凭据在设计使其变得相关时*标记*（§2），在会话设置时*收集*一次（§4），在那里绑定（`sessions.create()`）并被实际使用（冒烟测试）。请同时阅读 `shared/managed-agents-core.md`——其中有每个配置的详细说明；本文档是访谈脚本。

---

## 1. 描述任务

**以一个简短的引导和一个开放式问题开场——不要猜测，不要问卷式提问。** 用自己的话说明：

> Managed Agents 是托管式的——Anthropic 运行 Agent 循环、沙箱和基础设施；你只需定义 Agent。我们将分三步完成：Agent、它运行的环境，然后是一个实时测试会话。那么：描述你想要的 Agent——它应该做什么，什么触发它（一个人、一个事件、一个定时计划）？

让他们完整回答后再进行任何配置。

## 2. 配置 Agent——提议而非审问

他们的描述已经完成了访谈的工作。据此起草 Agent 配置，并**以提案形式呈现，内联你的建议**——用户对具体配置做出反应，而不是回答一系列问题。最多一个批次的追问来填补真正的空白。在描述提供切入点的地方给出建议：

- **工具**——默认启用全套预构建工具集（`agent_toolset_20260401`：`bash`、`read`、`write`、`edit`、`glob`、`grep`、`web_fetch`、`web_search`）。**建议 MCP 服务器**用于任务中提到的任何第三方服务（GitHub、Linear、Slack 等）——并在建议时标记每个服务所需的凭据（"Linear MCP → 启动时你需要一个 Linear API token"），这样 §4 的认证步骤只是走个流程，而非意外。收集本身等到 §4。仅当用户自己的应用必须响应调用时才使用自定义工具（名称、描述、输入 schema——处理代码由用户自行编写；不要生成它）。
- **技能**——当任务产出这类文件时，**建议**预构建的 `xlsx`/`docx`/`pptx`/`pdf` 技能；自定义技能通过 `skill_id`（每个 Agent 最多 20 个，预构建 + 自定义合计）。
- **Outcome**——如果描述暗示了可检查的"完成"标准（或者你可以在追问中引导出来：不是"一份好报告"而是"一个每个 SKU 都有数值 `price` 列的 CSV"），**建议使用 Outcome 启动**——框架根据评分标准进行评分和迭代（`shared/managed-agents-outcomes.md`）。
- **现有资源**——磁盘上的仓库（`github_repository`：URL，可选 `mount_path`/`checkout`；token 在 §4 中提供），要预填充的文件（Files API 上传 → `{type: "file", file_id, mount_path}`；只读），如果任务引用了它们。
- **模型**——默认 `claude-opus-4-8`；`claude-fable-5` 用于最困难的长周期任务（`shared/model-migration.md` → 迁移到 Claude Fable 5）。

> ‼️ **PR 创建还需要 GitHub MCP 服务器**——`github_repository` 挂载仅支持文件系统操作。在挂载中编辑 → 通过 `bash` 推送分支 → 通过 MCP 的 `create_pull_request` 工具打开 PR。

每个配置的详细说明：`shared/managed-agents-tools.md`（工具集、MCP、自定义工具、技能），`shared/managed-agents-environments.md`（仓库、文件）。

## 3. 环境

通常零到一个问题：

- **复用还是创建？** 环境在 Agent 之间共享——先检查是否存在现有环境。
- **网络**——默认不受限制的出站。仅在用户需要出站控制时切换到 `limited`——然后设置 `allow_mcp_servers: true` 或在 `allowed_hosts` 中列出每个 MCP 服务器域名，否则这些工具会静默失败。
- **建议 `self_hosted`**——当出现以下信号时：工具必须在自己的基础设施上运行、密钥不能离开基础设施、或需要云容器中没有的二进制文件/数据（`shared/managed-agents-self-hosted-sandboxes.md`；在 Claude Platform on AWS 上不可用）。否则使用 `cloud`——对于简单任务不要主动提及。

## 4. 会话——认证，然后测试运行

**认证在这里进行——收集 §2 中标记的凭据，此时配置已确定：** 一个 Vault（已有的或 `vaults.create()`）+ 为 §2 中声明的每个 MCP 服务器调用 `vaults.credentials.create()`、为任务使用的 API 密钥创建 `environment_variable` 凭据（在出站时替换；沙箱看到的是占位符），以及每个仓库挂载的 `authorization_token`。凭据是只写的；MCP 凭据通过 URL 匹配服务器并自动刷新。参见 `shared/managed-agents-tools.md` → Vault。

**静默可行性检查——在输出任何内容之前自行运行；仅暴露缺口。** 逐条检查任务：每个动词对应一个已启用的工具或 MCP 服务器（"打开 PR" → GitHub MCP，而不仅仅是挂载）；每个 MCP 服务器和仓库挂载都有认证步骤中的凭据；每个外部主机在网络选择下可达；任务引用的每个文件/仓库/数据集都已挂载；"完成"是可检查的。如果有缺失，说明并解决——不要输出一个你已经知道资源不足的配置。

**启动——选择其一，永远不要同时使用：**
- `user.message`——对话式。
- `user.define_outcome` + 评分标准——当 §2 确定了 Outcome 时使用；框架迭代并评分直到评分标准通过。
- **定时模式？** 完全跳过逐会话启动——创建**部署**（`deployments.create()` 配合 `schedule` + `initial_events`）；每次触发自动创建会话。参见 `shared/managed-agents-scheduled-deployments.md`。

需要融入运行时代码的机制：会话创建会阻塞直到资源挂载完成（错误的挂载在此处暴露，在 token 之前）；在发送启动消息*之前*打开事件流；在收到 `session.status_terminated` 或带有终止 `stop_reason` 的 `session.status_idle` 时中断——除了 `requires_action` 以外的任何情况（`shared/managed-agents-client-patterns.md` 模式 5）；使用量在 `span.model_request_end` 上到达；产出物在 `/mnt/session/outputs/` 中（`files.list({scope_id: session.id, ...})`）。

## 5. 集成——生成代码

直接从最后的回答到代码——不要前言，不要关于设置 vs. 运行时的讲座；两块结构已经展示了。生成**两个明确分离的代码块**：

**代码块 1——设置（运行一次，保存 ID）。** 优先使用 **YAML 文件 + `ant` CLI**——Agent 和环境是版本控制的定义，用户应将其签入并从 CI 中应用：

1. `<name>.agent.yaml`（扁平结构：`name`、`model`、`system`、`tools`、`mcp_servers`、`skills`）和 `<name>.environment.yaml`
2. ```sh
   AGENT_ID=$(ant beta:agents create < <name>.agent.yaml --transform id -r)
   ENV_ID=$(ant beta:environments create < <name>.environment.yaml --transform id -r)
   # CI 同步: ant beta:agents update --agent-id "$AGENT_ID" --version N < <name>.agent.yaml
   ```

如果用户要求则使用 SDK 备选方案——在 **Claude Platform on AWS 上是必需的**，因为认证使用 SigV4 而 `ant` CLI 没有 SigV4 模式（使用 `shared/claude-platform-on-aws.md` 中的平台客户端）：标记为 `# ONE-TIME SETUP — run once, save the IDs` 并调用 `environments.create()` → `agents.create()`。

> ⚠️ **部署比 MA 的其他部分更新。** 在生成 `ant beta:deployments …` 或 `client.beta.deployments` / `client.beta.deployment_runs` 调用之前，验证用户安装的 CLI/SDK 是否暴露了这些功能（`ant beta:deployments --help`；`hasattr(client.beta, "deployments")`）。如果没有，生成针对 `POST /v1/deployments` 的原始 HTTP 请求，携带 `managed-agents-2026-04-01` beta header（如果使用来自 `ant auth print-credentials` 的 Bearer token 认证，还需 `oauth-2025-04-20`），并留下升级说明标注哪些可以简化为 SDK 调用。

**定时模式？部署是设置，不是运行时。** 在代码块 1 中创建它，在 Agent/环境 ID 存在之后（`deployments.create()` 配合 `schedule` + `initial_events`）。代码块 2 则**不是**会话循环——没有逐次运行的启动消息可发送。改为生成：手动运行触发器（`POST /v1/deployments/{id}/run`），使用户可以立即测试而非等待首次触发——手动运行同时充当冒烟测试——加上一个获取辅助函数（最新的 `deployment_runs` 条目 → `session_id` → Console URL + `files.list(scope_id=session_id)` 获取产出物）。

**代码块 2——运行时（每次调用；对话式和 Outcome 模式）。** 使用检测到的语言的 SDK 代码（Python/TS/cURL——SKILL.md → 语言检测）；不要在这里生成 shell 循环：

1. 从配置/环境变量加载 `agent_id` + `env_id`
2. `sessions.create(agent=AGENT_ID, environment_id=ENV_ID, resources=[...], vault_ids=[...])`，然后打印 Console URL 以便用户实时观看：`https://platform.claude.com/workspaces/default/sessions/{session.id}`（将 `default` 替换为他们的工作区 slug）
3. **当任务依赖 MCP 服务器、凭据或受限主机时进行冒烟测试**——这些故障不会在 `sessions.create()` 时暴露，只在首次使用时暴露。一次低成本的探测轮次（"确认你可以访问 <服务> 并列出 1–2 个项目；不要开始任务"），验证后发送真正的启动消息。没有外部依赖时跳过。
4. 打开流 → 发送 §4 的启动消息 → 使用 §4 的终止条件循环。

> ⚠️ **永远不要在同一未受保护的代码块中同时生成 `agents.create()` 和 `sessions.create()`**——这会教用户每次运行都创建新 Agent，这是 #1 反模式。单脚本请求：将创建包装在 `if not os.getenv("AGENT_ID"):` 中。

从检测到的语言的 `{lang}/managed-agents/README.md` 中提取精确语法（cURL 和 C#：使用 `curl/managed-agents.md` 作为线路级参考）。不要发明字段名。
