---
name: chronicle
description: 分析 Copilot 会话历史记录，用于站会报告、使用技巧、会话搜索和会话重新索引。当用户请求站会、每日总结、使用技巧、工作流建议、想要按关键字/文件/PR 搜索或查找过去的会话、想要重新索引其会话存储，或询问删除会话数据时使用。
---

# Chronicle

使用 `copilot_sessionStoreSql` 工具分析用户的 Copilot 会话历史记录。此技能处理站会报告、使用分析、会话搜索和会话存储维护。

会话可能存储在本地（SQLite），并可选择同步到云端以实现跨设备访问。云端同步由 `chat.sessionSync.enabled` 设置控制。

**前提条件：** Chronicle 要求 `github.copilot.chat.localIndex.enabled` 设置为 `true`。如果 `copilot_sessionStoreSql` 工具不可用，请告知用户在 VS Code Settings 中启用此设置。

## 可用工具操作

`copilot_sessionStoreSql` 工具支持两种操作：

| 操作 | 用途 | `query` 参数 |
|------|------|-------------|
| `query` | 执行只读 SQL 查询 | 必填 |
| `reindex` | 重建本地会话索引 + 云端同步 | 不需要 |

## 工作流

### 站会（Standup）

当用户请求站会、每日总结或"我做了什么"时（例如 `/chronicle standup`）：

**第 1 步：收集过去 24 小时的活动**

使用 `copilot_sessionStoreSql`，设置 `action: "query"`，并遵循工具描述中所示的 SQL 方言（本地使用 SQLite，云端使用 DuckDB — 参见下方的 **Database Schema** 和 **Query Guidelines** 部分）。

查询 `sessions` 表，获取 `updated_at` 在过去 24 小时内的行，按 `updated_at` 降序排列。按后端划分的近期窗口谓词：

- **本地 SQLite**：`WHERE updated_at >= datetime('now', '-1 day')`
- **云端 DuckDB**：`WHERE updated_at >= now() - INTERVAL '1 day'`

然后，针对这些会话 ID，从 `session_refs` 中提取相关引用（PR、issue、commit）。如果需要某个特定会话的更多细节，再进一步查询 `turns`（以及 `session_files`，或云端的 `checkpoints`）— 不要一开始就转储每个会话的所有 turn。

如果在过去 24 小时内没有找到会话，请告知用户没有近期活动可报告，建议扩大时间窗口或执行 `/chronicle reindex`，然后停止。不要编造站会报告。

**第 2 步：包含没有 PR 的工作**

将每个近期会话视为候选工作项，即使它没有 PR、issue 或 commit 引用。PR 是辅助证据，而非事实来源。不要仅仅因为某个会话或分支没有 PR 就将其省略 — 使用会话摘要和 turn 内容来决定包含哪些内容。

**第 3 步：检查 PR 状态并格式化**

对于找到的任何 PR 引用，使用 GitHub CLI 或 MCP 工具检查当前状态（open、merged、draft、closed）。对于每个工作项，包含一行 PR 状态或"No PR found"行 — 绝不捏造 PR。

按工作流（分支/功能）分组格式化结果。严格使用以下结构：

```
Standup for <date>:

**✅ Done**

**Feature name** (`branch-name` branch, `repo-name`)
  - 3-7 words describing the status
  - Key files: 2-3 most important files changed
  - Merged: [#123](https://github.com/owner/repo/pull/123) or No PR found
  - Session: `full-session-id`

**🚧 In Progress**

**Feature name** (`branch-name` branch, `repo-name`)
  - 3-7 words describing the current state of work
  - Key files: 2-3 most important files being worked on
  - Draft: [#789](https://github.com/owner/repo/pull/789) or No PR found
  - Session: `full-session-id`
```

规则：
- 保持简洁 — 用户随时可以追问
- 使用 turn 数据（用户消息和助手回复）来了解完成了什么
- 使用 `session_files` 中的文件路径来确定哪些组件/区域受到了影响
- 将同一分支上的相关会话合并为一个条目
- 对于会话，每个功能/分支只显示最近的会话
- 使用 Markdown 链接语法链接 PR 和 issue
- 如果工作看起来已完成则归类为 Done，否则归类为 In Progress
- 如果会话没有分支或仓库，将其包含在"Other"部分下

### 技巧（Tips）

当用户请求技巧、工作流建议或如何改进时：

**第 1 步：调查用户的工作方式**

使用 `copilot_sessionStoreSql`，设置 `action: "query"`，探索他们的近期会话。目标是了解他们的模式 — 如何提示、使用什么工具、时间花在哪里。

要运行的查询（不要先解释你要做什么 — 立即开始查询）：
- 过去 7 天的会话：数量、持续时间、仓库
- Turn 数据：阅读实际用户消息以了解提示模式
- session_files：哪些文件和工具使用最频繁
- session_refs：PR/issue/commit 活动模式

**第 2 步：考虑可用功能**

如果当前工作区有 `.github/` 文件夹，检查 `.github/copilot-instructions.md`、`.github/skills/` 和 `.github/agents/`，看看存在哪些自定义配置。不要查看工作区之外的内容。寻找可用功能与用户实际使用之间的差距。

**第 3 步：提供技巧**

根据你了解到的信息，提供 3-5 个具体、可操作的建议。每个建议应该：
- 基于实际使用数据 — 引用你观察到的具体模式
- 非显而易见 — 跳过任何普通用户都已经知道的基本功能
- 关注差距 — 某个功能、工作流变更或不同方法能显著改善他们的体验

要探索的分析维度：
- **提示模式**：用户消息是模糊还是具体？他们是否提供了上下文？是否经常纠正或重定向 agent？
- **工具使用**：哪些工具使用最多？是否有未充分利用但可能有帮助的工具？
- **会话模式**：会话持续多长时间？是否有很多短暂的废弃会话？
- **文件模式**：代码库的哪些区域受到最多关注？是否有对同一文件的重复编辑？
- **工作流**：用户是否在使用 agent 模式、自定义指令、prompt 文件、技能？

如果会话存储数据很少，请承认这一点，并基于你在工作区中找到的配置建议一些可尝试的功能。

当推荐自定义技能、agent 或指令作为建议时，请查阅 **agent-customization** 技能以了解正确的文件创建模式 — 不要给出模糊的"创建自定义技能"建议而没有可操作的文件结构指导。

### 成本技巧（Cost Tips）

当用户请求成本技巧、减少 token 使用的方法或如何降低 Copilot 支出时（例如 `/chronicle cost-tips`）：

目标是**基于个人数据的、有数据支撑的建议**来减少 token 使用 — 而不是通用清单。每个建议必须指向你在其数据中观察到的具体模式。

**范围：聚焦于 VS Code 聊天会话**

其他 agent 界面（Copilot CLI、Copilot Coding Agent、Copilot Code Review、自定义 agent/subagent）的成本差异很大，会扭曲分析。默认情况下，**将每个查询过滤到交互式 VS Code 聊天界面**，以便发现仅反映该使用情况。只有当用户明确要求了解 CLI、Coding Agent 或自定义 agent 时才扩大范围 — 并且在这种情况下，按 agent 类型分别运行查询，而不是混合在一起。

存储的 `agent_name` 因后端而异 — **精确**匹配活跃后端的值（大小写和空格很重要）：

- **云端（DuckDB）**：`sessions.agent_name = 'VS Code Chat'`
- **本地（SQLite）**：`sessions.agent_name = 'GitHub Copilot Chat'`。本地还会将 subagent 调用（例如 `Explore`、`summarizeConversationHistory`）记录为独立的会话行；默认过滤器会正确排除它们。

简要检查一次 agent 混合情况，以便了解排除了什么（例如 `SELECT agent_name, COUNT(*) AS n FROM sessions WHERE updated_at > <30-day cutoff> GROUP BY 1 ORDER BY n DESC`）。如果交互式聊天的值在用户的会话中占少数，请在总结中提及这一点，以便用户知道建议仅针对其活动的一个子集，并**提供对另一个 agent 类型进行单独分析** — 列出你在混合检查中看到的候选项（例如"想对 `Copilot CLI` 或 `Copilot Coding Agent` 进行单独分析吗？"），以便用户知道可以扩大范围。

如果用户要求将范围扩大到特定界面（例如"现在做 CLI"、"给我的 Coding Agent 会话提供成本建议"、"包含我的 `Explore` subagent"），将默认的 `agent_name` 过滤器替换为请求的值，并**仅**针对该子集运行分析 — 不要在一次分析中混合界面。使用上述混合检查显示的精确 `agent_name` 字符串；跨后端的常见值包括 `Copilot CLI` / `copilotcli`、`Copilot Coding Agent`，以及任何自定义 agent / subagent 名称（例如 `Explore`、`summarizeConversationHistory`）。在总结中注明建议现在仅限于该界面，并指出在活跃后端上无法分析的内容（例如用户在本地时仅限云端的 token 列）。

**与成本相关的 schema（除下方 Database Schema 部分外）**

- **仅云端 DuckDB** — 本地 SQLite 存储**不**记录每个事件的 token 使用量，也没有 `events` 表。如果活跃后端是本地，则限制所有 token 查询，并告知用户真正的 token 级别分析需要启用云端同步（`chat.sessionSync.enabled`）。
- **events**（云端）：每个事件的计费 — `type = 'assistant.usage'` 的行包含 `usage_input_tokens`、`usage_output_tokens`、`usage_model`。将 `events e` JOIN 到 `sessions s ON s.id = e.session_id`，并过滤 `WHERE s.agent_name = 'VS Code Chat'` 以保持范围紧凑。
- **sessions.agent_name** / **agent_description**（两个后端）：交互式 VS Code 聊天界面在云端存储为 `'VS Code Chat'`，在本地存储为 `'GitHub Copilot Chat'`。其他值包括 `Copilot CLI` / `copilotcli`、`Copilot Coding Agent`、subagent（`Explore`、`summarizeConversationHistory`、`panel/editAgent` 等）以及自定义 agent。
- 在 `turns` 上使用 `LENGTH(user_message)`（或在 `events` 上使用 `LENGTH(user_content)`，其中 `type = 'user.message'`）来查找过大的粘贴内容。

**第 1 步：调查成本和 token 模式（仅限交互式 VS Code 聊天）**

使用 `copilot_sessionStoreSql`，设置 `action: "query"`。此步骤中的每个查询都必须将 `sessions.agent_name` 过滤为活跃后端对应的交互式 VS Code 聊天值 — 云端为 `'VS Code Chat'`，本地为 `'GitHub Copilot Chat'`。调查内容取决于活跃后端。

*云端（DuckDB）— 深入成本模式*（通过 `type = 'assistant.usage'` 过滤 `events` 行以获取计费行，并 JOIN `sessions` 以保持 `agent_name = 'VS Code Chat'`）：

- **Token 密集的会话和 turn** — 从 `events`（其中 `type = 'assistant.usage'`）中按会话和按模型汇总 `usage_input_tokens` 和 `usage_output_tokens`。哪些会话消耗了最多的 token？哪些模型？
- **输入输出比** — 当输入 token 远超输出 token 时，用户每个 turn 都在为重新发送臃肿的上下文付费。这是压缩、更小的工作集或新会话能有帮助的最强信号。
- **模型混合** — 按 `usage_model` 分解支出。是否将高级模型用于常规工作（重命名、简单编辑、状态检查），而这些工作本可以用更便宜的模型处理？
- **逐 turn 增长** — 在长会话中，`usage_input_tokens` 是否逐 turn 持续攀升？这是未使用压缩的强信号。
- **过大的粘贴** — 在 `events`（其中 `type = 'user.message'`）上使用 `LENGTH(user_content)` 来查找本应使用文件引用的用户消息（也可以在 `session_files` 中看到同一会话内对同一路径的重复读取）。

*本地（SQLite）— 无 token 数据；使用替代指标*（在每个查询上过滤 `sessions.agent_name = 'GitHub Copilot Chat'`）：

- **没有压缩的长会话** — 有很多 turn 但在 `checkpoints` 中没有行的会话（每个 `checkpoints` 行代表一次成功的压缩）。`LEFT JOIN checkpoints c ON c.session_id = s.id WHERE c.session_id IS NULL` 加上 turn 数阈值可以找出最佳候选。
- **延迟压缩** — 对于确实有 checkpoint 的会话，将 `checkpoints.checkpoint_number` 和 `created_at` 与会话的 turn 数进行比较。在一个 80 turn 会话的第 60 turn 才进行首次压缩，远不如在第 25 turn 时压缩有效。
- **重复的大文件读取** — 在 `session_files` 中，查找同一文件在一个会话内或跨会话被多次读取的情况。
- **工具调用抖动** — 有很多 turn 和重复工具调用的会话通常表明 agent 多次重新发现相同的上下文。
- **过大的粘贴** — 在 `turns` 上使用 `LENGTH(user_message)` 来查找本应使用文件引用的非常长的用户消息。

*两个后端：*

- **长时间运行的会话** — 有很多 turn 或跨越数小时的会话会在每个 turn 中拖拽不断增长的上下文窗口。
- **重复工作** — 同一文件/主题出现在多个会话中，或同一 agent 障碍反复出现（表明自定义技能、agent 或 `copilot-instructions.md` 条目可以让模型一次性完成工作）。
- **Subagent 使用** — 是否在主会话中运行重量级调查（让它们的 token 留在主上下文中），而本可以将它们委派给只返回摘要的 subagent？

深入几个成本最高的会话，阅读实际的对话 turn 来理解*为什么*它们成本高昂。不要只报告汇总数据 — 解释原因。

**第 2 步：将发现映射到功能和习惯**

如果当前工作区有 `.github/` 文件夹，检查 `.github/copilot-instructions.md`、`.github/skills/` 和 `.github/agents/`，看看已经存在哪些自定义配置。不要查看工作区之外的内容。与成本相关的能力需要牢记：

- 会话中途压缩（例如 `/compact`）以缩小上下文窗口；对于从不压缩的用户来说，这通常是最大的单一收益。
- 模型选择器 — 为常规工作切换到更便宜的模型；检查是否将高级模型用于简单任务。
- 开始新聊天而不是继续一个臃肿的会话。
- Subagent/委派，用于将繁重的研究卸载到一个子上下文中，其 token 不会累积到主会话中。
- 自定义技能（`.github/skills/`）和自定义 agent（`.github/agents/`），使重复的工作流不必每次都重新推导上下文。
- `.github/copilot-instructions.md` 用于编码项目约定，否则模型每次会话都需要被告知。
- 对于启用云端的用户，Copilot 使用视图可以查看当前的高级请求支出。

**第 3 步：提供技巧**

给用户提供 3-5 个具体、可操作的建议。每个建议应该：

- **基于他们的数据** — 引用你观察到的具体会话、文件、模型或模式（在有数据时提供大致数字：turn 数、token 总量、文件读取次数等）。
- **非显而易见** — 跳过任何回退用户都已经知道的基础知识。假设他们知道压缩和新聊天的存在；帮助他们注意到在重要的地方并没有*使用*它们。
- **尽可能量化收益** — "在那个 80 turn 会话的第 30 turn 左右进行压缩，将减少后续每个 turn 约 X 个输入 token"远比"考虑压缩"好得多。
- **具体** — 指出工作流变更、命令或配置文件编辑。如果建议是自定义技能或 agent，概述它将涵盖的内容。
- **保持在 VS Code 聊天范围内** — 建议应针对交互式 VS Code 聊天使用（压缩、模型选择器、新聊天、`.github/copilot-instructions.md`、自定义技能/agent、subagent 委派）。除非用户明确扩大了范围，否则不要提出 CLI 或 Coding Agent 特定的变更。

如果会话存储数据很少（例如云端存储为空，或只有少量本地交互式聊天会话），请直说并提供 2-3 个基于可用功能的非显而易见的省钱习惯，而不是捏造发现。如果用户使用仅本地存储，在结尾提及启用 `chat.sessionSync.enabled` 可以解锁每个事件的 token 分析，以便未来提供更精准的建议。

### 改进（Improve）

当用户要求基于会话历史改进其 agent 指令时（例如 `/chronicle improve`）：

**第 1 步：阅读当前指令文件**

阅读项目使用的指令文件（`.github/copilot-instructions.md` 或 `AGENTS.md`），了解已有内容。

如果文件**不存在**，你将创建它。在这种情况下，还需先分析代码库 — 查阅 **init** 技能并遵循其代码库探索方法。将该分析与第 2 步的会话历史发现相结合，生成一个全面的指令文件。

**第 2 步：调查会话历史**

使用 `copilot_sessionStoreSql` 进行探索。将所有查询限定在当前仓库或工作目录的会话范围内。

首先获取此仓库近期会话的概览，然后深入挖掘。你在寻找**摩擦** — agent 误解了什么或用户不得不纠正方向的信号：

- **纠正或重定向的用户消息** — 阅读可疑会话的实际对话 turn。寻找用户感到沮丧的领域。
- **开发循环困难** — agent 是否在测试、lint、构建或类型检查方面遇到困难？寻找重复失败的命令、测试重试或需要多次尝试才能解决的构建错误。
- **跨会话的模式** — 同类错误是否反复出现？

运用你的判断来决定运行哪些查询。当某些内容看起来有趣时，深入特定会话 — 阅读实际的逐 turn 对话以了解出了什么问题。

**第 3 步：呈现建议**

在呈现之前，查阅 **agent-customization** 技能以了解正确的文件约定、内容原则（链接而非嵌入、最小化、简洁）和反模式 — 这决定了建议应如何编写。

根据你的发现，简洁地呈现 3-5 个建议。解释你发现的问题以及自定义指令如何解决它。

关注项目特定的模式，而非通用建议。只建议那些解决数据中发现的、发生过多次的实际问题的指令。

在呈现所有建议后，询问用户想要应用哪些。然后仅对单个现有指令文件进行已批准的编辑（如果不存在则创建 `AGENTS.md`）。

### 搜索（Search）

当用户要求按关键字搜索、查找或检索过去的会话时（例如 `/chronicle search <query>`）：

**搜索策略**

1. 跨会话摘要、对话 turn（用户消息和助手回复）以及任何其他索引内容（checkpoint、文件路径、引用如 PR/issue/commit）进行搜索。用户的查询可能匹配主题、文件路径或 PR/issue 编号 — 覆盖所有三种情况。
2. 对于每个匹配的会话，收集足够的元数据来标记它：`s.id`、`s.repository`、`s.branch`、`s.summary`、`s.updated_at`，以及一个显示*为什么*匹配的简短片段（例如云端的 `substr(user_message, 1, 160)`，或匹配的 `file_path` / `ref_value`）。
3. 按仓库分组呈现结果，按最近更新时间排序。

**编写查询**

调用 `copilot_sessionStoreSql`，设置 `action: "query"` 和 `description: "Search sessions for <query>"`。遵循工具描述中所示的 SQL 方言（SQLite 与 DuckDB）。

Schema 要点（完整 schema 在下方 **Database Schema** 部分 — 编写查询前请重新阅读）：

- `sessions` 表的主键是 `id`（不是 `session_id`）。其他所有表使用 `session_id` 作为指向 `sessions.id` 的外键。始终从 `sessions` 中投影 `s.id`。
- 不要发明名称：没有 `started_at`（使用 `created_at`/`updated_at`），没有 `workspace`（本地使用 `cwd`；云端没有），没有 `title`（使用 `summary`），没有 `content`/`messages`（使用 `turns.user_message`/`turns.assistant_response`，或在云端使用 `events.user_content`/`events.assistant_content`）。
- **本地 SQLite**：优先使用 FTS5 `search_index` 表（`WHERE search_index MATCH '<query>'`）进行正文内容搜索 — `search_index` 已经有 `session_id` 列，所以直接选择它（`SELECT session_id, content FROM search_index WHERE search_index MATCH ...`）。不要将 `search_index.rowid` 与 `turns.rowid` 进行 JOIN；它们不相关，你会拉入不相关的会话。文件路径和引用不在 FTS 索引中 — 需要与 `session_files.file_path` 和 `session_refs.ref_value` 上的 `LIKE` 结合使用。
- **云端 DuckDB**：没有 FTS5 — 在 `sessions`、`turns`、`checkpoints`、`session_files`、`session_refs` 的文本列上使用 `ILIKE '%<query>%'`。

通过将用户查询中的单引号加倍来转义（`it's` → `it''s`）。对于多词 FTS5 查询，引用整个短语：`MATCH '"apply patch"'`。

**性能 — 避免云端超时**

在 `turns` 上使用 `ILIKE '%X%'` 是全表扫描。运行时间过长的云端查询会返回 `context deadline exceeded`。为保持在预算内：

- **使用两步模式**（推荐）：首先，使用窄时间窗口从 `turns` 中收集匹配的 `session_id`，并在 CTE 中使用 `GROUP BY session_id`。然后使用 `WHERE id IN (...)` 从 `sessions` 中充实数据。这避免了在大型扫描结果上进行昂贵的 JOIN，从而导致超时。
- 使用 `GROUP BY session_id` 和 `any_value()` / `MIN()` / `array_agg()` 按会话聚合匹配信息，而不是标量子查询或相关子查询。
- 在云端，默认对重型表使用 **7 天窗口**（在 `turns` 上使用 `WHERE timestamp >= now() - INTERVAL '7 days'`，在 `checkpoints` 上通过 `created_at` 同样处理）。如果没有返回结果，**逐步扩大**：7 天 → 30 天 → 90 天。在你的总结行中提及窗口，以便用户知道它是有界的。
- 在最终 SELECT 上保持 `LIMIT 50`。
- 如果查询超时，缩小窗口（而不是扩大）或去掉最重的表（通常是 `turns`），并告知用户你裁剪了什么。不要用相同的窗口重试 — 始终先缩小范围。

**输出格式**

对于每个会话，如果有 `summary` 则从中构建单行标签，否则使用返回的 `snippet`（截断到约 80 个字符）。绝不输出 `(no summary)`、`(no metadata)` 或裸会话 ID 列表。

如果你应用了时间窗口（例如云端默认的 7 天），在标题中包含它以便用户知道范围；否则省略范围短语或说"all time"。示例：

```
**Search results for "<query>"** (<n> sessions, <scope: e.g. "last 7 days" / "all time">)

_owner/repo_
- `session-id` — **<summary or snippet>**
  `branch` · updated <relative time> · matched in <match_kind>
- `session-id` — **<summary or snippet>**
  `branch` · updated <relative time> · matched in <match_kind>

_other-owner/other-repo_
- ...
```

规则：
- 始终每行渲染一个会话 — 绝不用逗号连接多个会话 ID。
- 标签优先使用 `summary`（如果非空）；否则使用片段；否则使用可用的后备值如匹配的文件路径。如果这些都为空，跳过该会话而不是显示"no summary"。
- 尽可能包含 `· matched in <match_kind>`（turn / file / ref / checkpoint / meta）—帮助用户了解每个会话为什么匹配。
- 按仓库分组。`repository` 为 NULL 的会话放在 _Other_ 标题下，格式相同（每行一个，带有可用标签）。
- 每个仓库的可见结果上限约为 10 个；如果更多，追加 `…and N more (refine your query)`。

**无结果**

如果没有返回任何行，请告知用户并建议：
- 尝试不同的关键字或更宽泛的搜索词（单个词，或子串而非短语）
- 扩大时间窗口（"搜索全部时间"、"包含更早的会话"）
- 如果尚未索引会话，运行 `/chronicle reindex`
- 运行 `/chronicle standup` 查看近期活动

### 重新索引（Reindex）

当用户要求重新索引、重建或刷新其会话存储时：

1. 调用 `copilot_sessionStoreSql`，设置 `action: "reindex"` 和 `description: "Reindex sessions"`。
2. 该工具从调试日志重建本地会话存储，如果启用了云端同步，则将新会话上传到云端。
3. 向用户展示前后统计数据和云端同步结果。

如果用户说"force reindex"或想要重新处理已索引的会话，在调用中添加 `force: true`。默认情况下，已索引的会话会被跳过以提高速度。

### 删除会话（Delete Sessions）

当用户要求删除会话数据或清除其历史记录时：

- 引导他们从命令面板运行 **Delete Session Sync Data** 命令（`github.copilot.sessionSync.deleteSessions`）。
- 该命令让他们选择要从本地存储和云端删除哪些会话。
- 工具本身不支持删除 — 这是为了防止意外数据丢失而有意设计的。

## 查询指南

使用 `action: "query"` 时：
- 每次调用只执行一个查询 — 不要用分号组合多个语句
- 只允许 `SELECT` 和 `WITH`（CTE）。`DESCRIBE`、`SHOW`、`PRAGMA` 和任何变更语句都被阻止 — 重新阅读下方 **Database Schema** 部分，而不是尝试内省
- 始终使用 LIMIT（最大 100），优先使用聚合（COUNT、GROUP BY）而非原始行转储
- 查询 **turns** 表获取对话内容 — 它能提供最丰富的洞察
- 查询 **session_files** 获取文件路径和工具使用模式
- 查询 **session_refs** 获取 PR/issue/commit 链接
- 使用 session_id 进行表连接以实现完整分析
- 始终基于 **updated_at**（而非 created_at）进行时间范围过滤
- 始终将 sessions 与 turns 进行 JOIN 以获取会话内容 — 不要仅依赖 sessions.summary

### 查询路由

该工具根据用户的云端同步设置自动路由查询：
- **云端已启用**：查询发送到云端 DuckDB 后端，其中包含跨所有设备和 agent（VS Code、CLI、Copilot Coding Agent、PR review）的全部会话。工具描述将显示 DuckDB SQL 语法 — 遵循它。
- **云端已禁用**：查询发送到本地 SQLite，其中仅包含此设备的会话。工具描述将显示 SQLite 语法。

工具的描述会根据活跃后端动态变化。**始终遵循工具描述中显示的 SQL 语法** — 它匹配活跃后端。

## Database Schema

### 表（本地和云端共有，除非另有说明）

- **sessions**：id、cwd（工作区文件夹路径 — 在云端始终为 NULL）、repository、branch、host_type、summary、agent_name、agent_description、created_at、updated_at
- **turns**：session_id、turn_index、user_message、assistant_response（前约 1000 个字符，可能被截断）、timestamp
- **checkpoints**：session_id、checkpoint_number、title、overview、history、work_done、technical_details、important_files、next_steps、created_at — 存储压缩状态的压缩检查点。注意：云端的列更少（没有 history/work_done/technical_details）。
- **session_files**：session_id、file_path、tool_name、turn_index、first_seen_at
- **session_refs**：session_id、ref_type（commit/pr/issue）、ref_value、turn_index、created_at
- **search_index**：FTS5 虚拟表（仅本地）。列：`content`、`session_id`、`source_type`（`turn`/`assistant`/`checkpoint`/等）、`source_id`。使用 `WHERE search_index MATCH 'query'` 进行全文搜索，并直接投影 `session_id` — **不要**将 `search_index.rowid` 与 `turns.rowid` 进行 JOIN（rowid 是独立的，JOIN 会匹配错误的行）。使用 `snippet(search_index, 0, '[', ']', '…', 12)` 或 `substr(content, 1, 160)` 获取片段。

### 仅云端表

- **events**：原始事件表（约 90 列）。关键列：session_id、timestamp、type、user_content、assistant_content、tool_start_name、tool_complete_success、tool_complete_result_content、usage_model、usage_input_tokens、usage_output_tokens
- **tool_requests**：session_id、tool_call_id、name、arguments_json

日期计算（SQLite）：`datetime('now', '-1 day')`、`datetime('now', '-7 days')`
日期计算（云端/DuckDB）：`now() - INTERVAL '1 day'`、`now() - INTERVAL '7 days'`。使用 `ILIKE` 进行文本搜索（没有 FTS5/MATCH），使用 `date_diff('minute', start, end)` 计算持续时间。