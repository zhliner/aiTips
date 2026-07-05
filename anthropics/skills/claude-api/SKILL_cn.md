---
name: claude-api
description: |-
  Claude API / Anthropic SDK 参考资料 — 模型 ID、定价、参数、流式传输、工具调用、MCP、代理、缓存、token 计数、模型迁移。
  TRIGGER — 在打开目标文件之前先阅读；不要因为它“看起来只是一个很短的文件”就跳过 —— 任何情况下：提示词以 Claude/Anthropic 形式命名（Claude、Anthropic、Fable、Opus、Sonnet、Haiku、`anthropic`、`@anthropic-ai`、`claude-*`、`us.anthropic.*`、`[1m]`）；用户询问有关 LLM（价格/模型选择/限制/缓存）——永远不要凭记忆回答；或者任务是 LLM 形态但提供商未说明（agent/MCP/tool-definition/multi-agent/RAG/LLM-judge/computer-use; generate/summarize/extract/classify/rewrite/converse over NL; debugging refusals/cutoffs/streaming/tool-calls/tokens）。
  仅在另一个提供商正在处理时跳过（覆盖所有触发条件）：查询中提到 OpenAI/GPT/Gemini/Llama/Mistral/Cohere/Ollama；或者项目中先执行 `grep -rE 'openai|langchain_openai|google.generativeai|genai|mistralai|cohere|ollama'`（如果没有指定提供商，先执行这个 grep，再阅读文件）。
license: Complete terms in LICENSE.txt
---

# 使用 Claude 构建由 LLM 驱动的应用

这项技能可帮助你使用 Claude 构建由 LLM 驱动的应用。根据你的需求选择合适的接入层，识别项目语言，然后阅读与之对应的语言特定文档。

## 开始之前

扫描目标文件（如果没有目标文件，则扫描提示词和项目）中是否存在非 Anthropic 提供商标记：`import openai`、`from openai`、`langchain_openai`、`OpenAI(`、`gpt-4`、`gpt-5`，以及类似 `agent-openai.py` 或 `*-generic.py` 的文件名，或者任何明确要求保持代码对提供商中立的说明。如果你发现这些内容，请停止并告诉用户：这个技能会生成 Claude/Anthropic SDK 代码；询问他们是想把文件切换为 Claude，还是想要一个非 Claude 的实现。不要用 Anthropic SDK 调用修改非 Anthropic 文件。

## 输出要求

当用户要求你添加、修改或实现 Claude 功能时，你的代码必须通过以下方式之一调用 Claude：

1. **官方 Anthropic SDK**（针对项目语言的 `anthropic`、`@anthropic-ai/sdk`、`com.anthropic.*` 等）。这是默认选择，只要项目支持对应 SDK。
2. **原始 HTTP**（`curl`、`requests`、`fetch`、`httpx` 等）——仅当用户明确要求使用 cURL/REST/原始 HTTP、项目本身就是 shell/cURL 项目，或者该语言没有官方 SDK 时才使用。

不要混用两者——不要因为觉得更轻量，就在 Python 或 TypeScript 项目中使用 `requests`/`fetch`。不要退回到 OpenAI 兼容 shim。

**永远不要猜测 SDK 的使用方式。** 函数名、类名、命名空间、方法签名和导入路径必须来自明确的文档——要么来自这个技能中的 `{lang}/` 文件，要么来自 `shared/live-sources.md` 中列出的官方 SDK 仓库或文档链接。如果你需要的绑定没有在技能文件中明确记录，请先通过 `shared/live-sources.md` 对相关 SDK 仓库进行 WebFetch，然后再编写代码。不要凭借 cURL 形态或其他语言 SDK 的经验去推断 Ruby/Java/Go/PHP/C# 的 API。

**如果 WebFetch 或仓库访问失败**（网络受限、超时、clone 被阻止）：不要继续反复重试——从 `{lang}/` 文件中的模式、命名空间/包表中写出代码，然后在编译器或解释器上运行它，并根据报错输出进行迭代。对于静态类型 SDK（C#、Java、Go），与本地错误进行编译修复循环，通常比被网络研究卡住更快。

## 默认设置

除非用户另有要求：

对于 Claude 模型版本，请默认使用 Claude Opus 4.8，可通过精确模型字符串 `claude-opus-4-8` 访问。对于任何相对复杂的任务，请默认启用自适应思考（`thinking: {type: "adaptive"}`）。最后，对于任何可能涉及长输入、长输出或高 `max_tokens` 的请求，请默认启用流式传输——这样可以避免请求超时。如果你不需要处理单独的流事件，可以使用 SDK 的 `.get_final_message()` / `.finalMessage()` 辅助方法来获取完整响应。

## ⚠️ API 漂移 — 你的训练数据可能已过时

多个常见的 Claude API 形态在 2025–2026 年发生了变化。如果你记得某个模式，请在编写代码前先对照技能中的 `{lang}/` 文件进行验证——下面这些是最常见的漂移点：

| 领域 | 旧式写法 | 当前 API |
|---|---|---|
| 扩展思考 | `thinking: {type: "enabled", budget_tokens: N}` | 在 Claude 4.6+ 模型上使用 `thinking: {type: "adaptive"}`。`budget_tokens` 在 Opus 4.6 / Sonnet 4.6 上已弃用，并且在 Fable 5 / Sonnet 5 / Opus 4.8 / 4.7 上会被以 400 错误拒绝。4.6 之前的模型仍使用 `budget_tokens`。 |
| Web 搜索 / web fetch 工具类型 | `web_search_20250305`、`web_fetch_20250910` | 在 Opus 4.8/4.7/4.6、Sonnet 5 和 Sonnet 4.6 上使用 `web_search_20260209`、`web_fetch_20260209`（动态过滤）。较旧模型保留基础变体；在 Vertex AI 上只有基础 `web_search_20250305` 可用（Vertex 上没有 web fetch）——见下文的 Server Tools 二维码。 |
| PHP 参数名 | 命名参数使用 snake_case 线下名（如 `max_tokens`） | 顶层命名参数使用 camelCase（如 `maxTokens`）。嵌套数组键因功能而异（如 `'taskBudget'`、`'skillID'`、`'mcp_server_name'`）——请精确复制文档示例中的键；不要批量转换。 |

这个技能中的 `{lang}/` 文件优先于你记忆中的旧模式。

---

## 子命令

如果提示词底部的用户请求是一个裸子命令字符串（没有正文），请搜索本文档中的所有“Subcommands”表格——包括下面追加的任何章节——并直接按照对应的 Action 列执行。这让用户可以通过 `/claude-api <subcommand>` 调用特定流程。如果没有任何表格匹配，则把请求当作普通正文处理。

| 子命令 | 动作 |
|---|---|
| `migrate` | 将现有 Claude API 代码迁移到更新的模型。**立即阅读 `shared/model-migration.md`**，按顺序执行：步骤 0（确认范围——在任何编辑前先询问哪些文件/目录），步骤 1（对每个文件分类），然后是每个目标的 breaking-changes 部分。不要总结指南——直接执行。如果用户没有指定目标模型，请在同一轮中询问要迁移到哪个模型。 |

---

## 语言检测

在阅读代码示例前，先判断用户正在使用哪种语言：

1. **查看项目文件**，推断语言：

   - `*.py`、`requirements.txt`、`pyproject.toml`、`setup.py`、`Pipfile` → **Python** — 从 `python/` 读取
   - `*.ts`、`*.tsx`、`package.json`、`tsconfig.json` → **TypeScript** — 从 `typescript/` 读取
   - `*.js`、`*.jsx`（当前不存在 `.ts` 文件）→ **TypeScript** — JS 使用相同 SDK，从 `typescript/` 读取
   - `*.java`、`pom.xml`、`build.gradle` → **Java** — 从 `java/` 读取
   - `*.kt`、`*.kts`、`build.gradle.kts` → **Java** — Kotlin 使用 Java SDK，从 `java/` 读取
   - `*.scala`、`build.sbt` → **Java** — Scala 使用 Java SDK，从 `java/` 读取
   - `*.go`、`go.mod` → **Go** — 从 `go/` 读取
   - `*.rb`、`Gemfile` → **Ruby** — 从 `ruby/` 读取
   - `*.cs`、`*.csproj` → **C#** — 从 `csharp/` 读取
   - `*.php`、`composer.json` → **PHP** — 从 `php/` 读取

2. **如果检测到多种语言**（例如同时有 Python 和 TypeScript 文件）：

   - 检查用户当前文件或问题与哪种语言更相关
   - 如果仍然不明确，就询问：“我检测到同时存在 Python 和 TypeScript 文件。你要为哪个语言实现 Claude API 集成？”

3. **如果无法推断语言**（空项目、没有源文件，或不支持的语言）：

   - 使用 AskUserQuestion 并给出选项：Python、TypeScript、Java、Go、Ruby、cURL/raw HTTP、C#、PHP
   - 如果 AskUserQuestion 不可用，则默认显示 Python 示例，并注明：“正在展示 Python 示例。如果你需要其他语言，请告诉我。”

4. **如果检测到不支持的语言**（Rust、Swift、C++、Elixir 等）：

   - 建议使用 `curl/` 中的 cURL/raw HTTP 示例，并说明社区 SDK 可能存在
   - 也可以提供 Python 或 TypeScript 示例作为参考实现

5. **如果用户需要 cURL/raw HTTP 示例**，则从 `curl/` 读取。

### 语言特定功能支持

| 语言 | Tool Runner | Managed Agents | 说明 |
| ---------- | ----------- | -------------- | ------------------------------------- |
| Python | Yes (beta) | Yes (beta) | 完整支持 — `@beta_tool` 装饰器 |
| TypeScript | Yes (beta) | Yes (beta) | 完整支持 — `betaZodTool` + Zod |
| Java | Yes (beta) | Yes (beta) | 带注解类的 Beta 工具使用 |
| Go | Yes (beta) | Yes (beta) | `toolrunner` 包中的 `BetaToolRunner` |
| Ruby | Yes (beta) | Yes (beta) | `BaseTool` + `tool_runner` in beta |
| C# | Yes (beta) | Yes (beta) | `BetaToolRunner` + 原始 JSON schema |
| PHP | Yes (beta) | Yes (beta) | `BetaRunnableTool` + `toolRunner()` |
| cURL | N/A | Yes (beta) | 原始 HTTP，不使用 SDK 功能 |

> **Managed Agents 代码示例**：为 Python、TypeScript、Go、Ruby、PHP、Java 和 cURL 提供了专门的语言 README（`{lang}/managed-agents/README.md`、`curl/managed-agents.md`）。请先阅读你的语言对应 README，再阅读通用的 `shared/managed-agents-*.md` 概念文件。**Agents 是持久化的 — 创建一次，之后通过 ID 引用。** 请保存 `agents.create` 返回的 agent ID，并将其传给后续的每个 `sessions.create`；不要在请求路径中重复调用 `agents.create`。Anthropic CLI（`ant`）是从版本控制的 YAML 创建 agent 和环境的一种便捷方式——见 `shared/anthropic-cli.md`。如果你需要的绑定没有在 README 中展示，请通过 `shared/live-sources.md` 的相关条目进行 WebFetch，而不要猜测。C# 通过 `client.Beta.Agents` 及其相关命名空间提供 Beta Managed Agents 支持。

---

## 我应该使用哪种接入层？

> **先从简单开始。** 默认选择满足需求的最简单层级。大多数场景只需单次 API 调用和工作流；只有当任务确实需要开放式、由模型主导的探索时，才需要进入 agent。

| 用例 | 层级 | 推荐接入层 | 原因 |
| ----------------------------------------------- | --------------- | ------------------------- | ------------------------------------------------------------ |
| 分类、摘要、提取、问答 | Single LLM call | **Claude API** | 一次请求，一次响应 |
| 批量处理或 embedding | Single LLM call | **Claude API** | 专用端点 |
| 具有代码控制逻辑的多步骤管道 | Workflow | **Claude API + tool use** | 你来编排循环 |
| 带你自己的工具的自定义 agent | Agent | **Claude API + tool use** | 最大灵活性 |
| 由服务器托管状态化 agent，且带工作区 | Agent | **Managed Agents** | Anthropic 运行循环并托管工具执行沙盒 |
| 持久化、版本化的 agent 配置 | Agent | **Managed Agents** | Agents 是存储对象；sessions 会绑定到某个版本 |
| 长时间多轮 agent，带文件挂载 | Agent | **Managed Agents** | 每个 session 的容器、SSE 事件流、Skills + MCP |

> **说明：** 当你希望 Anthropic 运行 agent 循环，并托管工具执行的容器（文件操作、bash、代码执行）时，Managed Agents 是正确选择。如果你想自己托管计算，或者运行你自己的自定义工具运行时，Claude API + tool use 才是合适的选择——可以使用 tool runner 自动处理循环，或者使用手动循环以获得更细粒度控制（审批门控、自定义日志、条件执行）。

> **云提供商访问。** **Claude Platform on AWS** 由 Anthropic 运维，拥有与 API 同步的功能——见 `shared/claude-platform-on-aws.md` 获取客户端设置。关于 **Claude Platform on AWS**、**Amazon Bedrock**、**Google Vertex AI** 和 **Microsoft Foundry** 的特性可用性，请见 `shared/platform-availability.md`——该表格是本技能里的唯一事实来源；不要从其他地方推断可用性。

### 决策树

```
你的应用需要什么？

0. 哪个提供商？
   ├── 第一方 API 或 Claude Platform on AWS → 继续（完整功能可用；具体例外见 shared/platform-availability.md）。
   └── Amazon Bedrock、Google Vertex AI 或 Microsoft Foundry → Claude API（+ agents 的 tool use）；具体支持情况见 shared/platform-availability.md。

1. 单次 LLM 调用（分类、摘要、提取、问答）
   └── Claude API — 一次请求，一次响应

2. 你是否希望 Anthropic 运行 agent 循环，并托管每个 session 的容器，让 Claude 执行工具（bash、文件操作、代码）？
   └── 是 → Managed Agents — 服务器托管 session、持久化 agent 配置、
       SSE 事件流、Skills + MCP、文件挂载。
       示例：“每个任务都有独立工作区的有状态编码 agent”、
             “将事件流输出到 UI 的长时间研究 agent”、
             “跨多个 session 持久化、版本化配置的 agent”

3. Workflow（多步骤、由代码编排、带你自己的工具）
   └── 带 tool use 的 Claude API — 由你控制循环

4. 开放式 agent（模型决定自己的轨迹、你自己的工具、你托管计算）
   └── Claude API agentic loop（最大灵活性）
```

### 我应该构建 agent 吗？

在选择 agent 层级之前，请检查以下四个标准：

- **复杂度** — 任务是否多步骤，且难以提前完整定义？（例如“把这个设计文档变成 PR” vs. “从这个 PDF 中提取标题”）
- **价值** — 结果是否值得更高的成本和延迟？
- **可行性** — Claude 是否适合这类任务？
- **出错成本** — 是否容易捕获并恢复错误？（测试、审查、回滚）

如果对其中任何一项回答是“否”，则停留在更简单的层级（单次调用或 workflow）。

---

## 架构

所有流程都经过 `POST /v1/messages`。工具和输出约束都是这个单一端点的功能，而不是单独的 API。

**用户定义工具** — 你定义工具（通过装饰器、Zod schema 或原始 JSON），SDK 的 tool runner 会负责调用 API、执行你的函数，并循环直到 Claude 完成。若要完全控制，你也可以手动编写循环。

**服务端工具** — Anthropic 托管的工具，在 Anthropic 的基础设施上运行。代码执行是完全服务端的（在 `tools` 里声明，Claude 会自动运行代码）。计算机使用可以是服务端托管，也可以是自托管。

**结构化输出** — 约束 Messages API 的响应格式（`output_config.format`）和/或工具参数校验（`strict: true`）。推荐方法是 `client.messages.parse()`，它会自动根据你的 schema 校验响应。注意：旧的 `output_format` 参数已弃用；请在 `messages.create()` 上使用 `output_config: {format: {...}}`。

**支持的其他端点** — 批处理（`POST /v1/messages/batches`）、文件（`POST /v1/files`）、Token 计数（`POST /v1/messages/count_tokens` — 见 `shared/token-counting.md`）、模型（`GET /v1/models`、`GET /v1/models/{id}` — 用于实时发现能力和上下文窗口）都支持或辅助 Messages API 请求。

---

## 当前模型（缓存时间：2026-06-24）

| 模型 | 模型 ID | 上下文 | 输入 $/1M | 输出 $/1M |
| ----------------- | ------------------- | -------------- | ---------- | ----------- |
| Claude Fable 5 | `claude-fable-5` | 1M | $10.00 | $50.00 |
| Claude Mythos 5（仅 Project Glasswing） | `claude-mythos-5` | 1M | $10.00 | $50.00 |
| Claude Opus 4.8 | `claude-opus-4-8` | 1M | $5.00 | $25.00 |
| Claude Opus 4.7 | `claude-opus-4-7` | 1M | $5.00 | $25.00 |
| Claude Opus 4.6 | `claude-opus-4-6` | 1M | $5.00 | $25.00 |
| Claude Sonnet 5 | `claude-sonnet-5` | 1M | $3.00（2026-08-31 前为 $2.00 intro） | $15.00（intro 为 $10.00） |
| Claude Sonnet 4.6 | `claude-sonnet-4-6` | 1M | $3.00 | $15.00 |
| Claude Haiku 4.5 | `claude-haiku-4-5` | 200K | $1.00 | $5.00 |

**始终使用 `claude-opus-4-8`，除非用户明确指定了其他模型。** 这是强制要求。不要使用 `claude-sonnet-5`、`claude-sonnet-4-6` 或其他模型，除非用户明确说“use sonnet”或“use haiku”。不要为了省钱降级——那是用户的决定，不是你的。只有在用户明确要求 Claude Fable 5、“fable” 或 Anthropic 最强模型时才使用 `claude-fable-5`——它与 Opus 家族具有不同的 API 行为（见下文），且价格高于 Opus 层级。

### Claude Fable 5（`claude-fable-5`）— 最强的广泛发布模型

Claude Fable 5 是 Anthropic 最强的广泛发布模型，适用于最苛刻的推理和长周期 agentic 工作。**Claude Mythos 5**（`claude-mythos-5`）通过 Project Glasswing 提供相同能力、相同价格和相同 API 表面（参与是访问方式唯一要求），它接替了仅邀请制的 Claude Mythos Preview（`claude-mythos-preview`）——下面所有内容同时适用于这两个模型。1M 上下文窗口（也是默认值），最大输出 128K。与 Opus 层级的关键 API 差异——详情见 `shared/model-migration.md` → Migrating to Claude Fable 5：

- **思考始终开启** — 省略 `thinking` 参数（或者发送 `{type: "adaptive"}`）。任何其他显式配置都会被拒绝：`{type: "disabled"}` 和 `{type: "enabled", budget_tokens: N}` 都会返回 400。通过 `output_config.effort` 控制深度（支持 `low` 到 `xhigh` 与 `max`）。
- **原始 chain of thought 永远不会返回** — 响应会携带常规 `thinking` 块（不是 `redacted_thinking`）：`display: "summarized"` 会返回可读摘要，`"omitted"`（默认）则让 `thinking` 字段为空字符串。回放规则：在相同模型上将 thinking 块原样回传（包括空文本块——API 拒绝的是“被修改的”块，而不是已读取的块）；而不同模型会将它们从提示词中丢弃（通常无报错——丢弃发生在计费前，所以不会被计费，也没有什么需要剥离）。其他模型的常规 thinking 块可以跨模型自由回放。
- **分词器** — 与 Opus 4.8 相同（Opus 4.7 引入）。从 Opus 4.7/4.8 迁移时 token 数量大致不变；每 token 价格不同。从 Opus 4.6、Sonnet、Haiku 或更旧模型迁移时，请用 `count_tokens` 重新基线。
- **`refusal` 停止原因 — 要处理，并默认开启 fallback** — 安全分类器可能会拒绝请求（HTTP 200、`stop_reason: "refusal"`，并附带 `stop_details` 类别）。预输出拒绝时 `content` 为空数组，且完全不计费；中途流式拒绝时会把已经流出的输出计费——请丢弃部分输出。在读取 `content` 前务必检查 `stop_reason`。恢复是 **API 上的可选行为**：大多数 Claude 消费端表层都会内置 Claude Opus 4.8 fallback，但没有开启 opt-in 的 API 请求只会在拒绝时直接停止——并且对良性相邻工作（安全工具、生命科学任务）也会出现误判。**当你编写 `claude-fable-5` 代码时，请默认包含服务端 `fallbacks` 参数**（`betas: ["server-side-fallback-2026-06-01"]` + `fallbacks: [{"model": "claude-opus-4-8"}]`；适用于 Claude API 和 Claude Platform on AWS）：被拒绝的请求会在同一调用内由 fallback 模型透明重试，并自动按信用式重定价（在任何输出前拒绝时不计费；救援则按 fallback 模型费率计费）。告诉用户你已经启用了此项；只有当他们拒绝时才移除它。GA SDK 的客户端 `BetaRefusalFallbackMiddleware` + `BetaFallbackState` 会在服务端 fallback 不可用的场景（包括 Amazon Bedrock、Vertex AI、Microsoft Foundry）中处理重试；fallback 的信用会退回客户端重试切换缓存的成本。代码示例：你的语言对应的 claude-api 文档中的 Refusal Fallbacks 部分；完整语义见迁移指南中的 refusal 部分。
- **没有 assistant prefill** — 与 4.6+ 家族其余模型一致。
- **需要 30 天数据保留** — Claude Fable 5 不能在零数据保留条件下使用；组织若未满足保留配置要求，请求会返回 `400 invalid_request_error`。
- **更长的对话、更不同的提示词** — 在复杂任务上，单次请求可能要跑很久（计划超时/流式/进度 UX）；effort 扫描应包含 `low`/`medium` 以适配常规工作；为旧模型编写的提示词通常过于规定，反而降低输出质量。见 `shared/model-migration.md` → Migrating to Claude Fable 5 → Behavioral shifts (prompt-tunable) 的推荐提示词片段（anti-overplanning、no-tidying、grounded progress claims、boundaries、async sub-agents、memory、`send_to_user`）。

**关键：只使用上表中的精确模型 ID 字符串——它们是完整且现成的。不要附加日期后缀。** 例如使用 `claude-sonnet-4-6`，不要使用 `claude-sonnet-4-6-20251114` 或你从训练数据中记忆到的任何其他日期后缀。若用户请求了上表中不存在的旧模型（例如“opus 4.5”“sonnet 3.7”），请读取 `shared/models.md` 获取准确 ID——不要自己构造。

说明：如果你对上面某些模型字符串不熟悉，这很正常——那通常意味着它们是在你的训练数据截止时间之后发布的。请放心，它们是真实模型；我们不会拿你开玩笑。

**实时能力查询：** 上表是缓存数据。当用户询问“X 的上下文窗口是多少”“X 是否支持 vision/thinking/effort”“哪些模型支持 Y”时，请查询 Models API（`client.models.retrieve(id)` / `client.models.list()`）——见 `shared/models.md` 的字段参考和能力过滤示例。

---

## 身份验证（快速参考）

**未设置 `ANTHROPIC_API_KEY` 并不意味着没有凭据。** SDK 和 `ant` CLI 会按以下顺序解析凭据（先匹配优先）：`ANTHROPIC_API_KEY` → `ANTHROPIC_AUTH_TOKEN` → `ant auth login` 选择或激活的 OAuth profile → Workload Identity Federation 环境变量 → 磁盘上的默认 profile。仅在 `ant auth login` 后，裸的 `Anthropic()` / `new Anthropic()` / `anthropic.NewClient()` 就能在不设置环境变量的情况下工作。

**当你需要调用 API 且 `ANTHROPIC_API_KEY` 未设置时，不要直接向用户要 key。** 先运行 `ant auth status`——它会显示当前激活的凭据来源和 profile。如果它报告有激活 profile：

- **SDK 代码或 `ant` CLI：** 直接运行即可。零参数构造函数和所有 `ant …` 子命令都会自动读取 profile——不需要环境变量。
- **原始 `curl` / HTTP：** 通过 `ant auth print-credentials --access-token` 获取短时 token，并作为 `Authorization: Bearer <token>` 发送，同时附带头部 `anthropic-beta: oauth-2025-04-20`（OAuth token 放在 `Authorization: Bearer`，不是 `x-api-key:`——把 curl 从 API key 版本改写，本质上是改 header，不是换 key）。务必传 `--access-token`；不带参数的形式会输出 JSON，而不是裸 token。

只有当 `ant auth status` 显示没有激活的凭据来源（或 `ant` 本身未安装）时，才向用户索要 key。建议先使用 `ant auth login`——它会在 `~/.config/anthropic/` 下存储 profile，SDK 会自动读取——或者使用导出的 `ANTHROPIC_API_KEY` 作为替代方案。

完整的认证细节（命名 profile、scope、API key 覆盖 profile 的陷阱、refresh token 失效）：`shared/anthropic-cli.md`。

---

## 思考与 Effort（快速参考）

**Fable 5 / Opus 4.8 / 4.7 / Sonnet 5 — 仅自适应思考：** 使用 `thinking: {type: "adaptive"}`。`thinking: {type: "enabled", budget_tokens: N}` 会返回 400——自适应是唯一的开启模式。在 Opus 4.8、Opus 4.7 和 Sonnet 5 上，`{type: "disabled"}` 和省略 `thinking` 都可以（Sonnet 5 上省略时会运行自适应；Opus 4.7/4.8 上省略时则不会思考——显式设置 `{type: "adaptive"}`）；在 Fable 5 上，显式 `{type: "disabled"}` 会返回 400——改为直接省略 `thinking` 参数。采样参数（`temperature`、`top_p`、`top_k`）也会被移除，并会 400。Opus 4.8 保留与 4.7 相同的请求表面，没有新的 breaking change——见 `shared/model-migration.md` → Migrating to Opus 4.8 的行为重调优，以及 → Migrating to Opus 4.7 的完整 breaking-change 清单（适用于从 4.6 或更早版本迁移时）。注意：在关闭 `thinking` 时，Opus 4.8 可能会把更长的推理写入可见响应——请保持自适应思考开启，或者增加“仅输出最终答案”的指令（见迁移指南）。
**Opus 4.6 — 推荐使用自适应思考：** 使用 `thinking: {type: "adaptive"}`。Claude 会动态判断何时以及思考多少。无需 `budget_tokens`——`budget_tokens` 在 Opus 4.6 和 Sonnet 4.6 上已弃用，不应该用于新代码。自适应思考也会自动启用交错思考（无需 beta header）。**当用户要求“extended thinking”“thinking budget”或 `budget_tokens` 时，请始终使用 Fable 5、Opus 4.8、4.7 或 4.6，并配合 `thinking: {type: "adaptive"}`。固定思考 token budget 的概念已弃用——自适应思考已经取代它。请不要为新的 4.6/4.7/4.8 代码使用 `budget_tokens`，也不要降级到更旧模型。** *渐进迁移例外：* `budget_tokens` 在 Opus 4.6 和 Sonnet 4.6 上仍可作为过渡性逃生口——如果你正在迁移现有代码，需要在你调整 `effort` 之前设置一个硬性的 token 上限，请见 `shared/model-migration.md` → Transitional escape hatch。注意：这一例外不适用于 Fable 5、Opus 4.7 或 4.8——那里 `budget_tokens` 已被完全移除。
**Effort 参数（GA，无 beta header）：** 通过 `output_config: {effort: "low"|"medium"|"high"|"max"}` 控制思考深度和整体 token 消耗（位于 `output_config` 内，不是顶层）。默认值是 `high`（等价于省略）。`max` 在 Fable 5、Opus 4.6 及之后、Sonnet 5 和 Sonnet 4.6 上可用（Haiku 或更早的 Sonnet 不可）。Opus 4.7 新增了 `"xhigh"`（位于 `high` 和 `max` 之间）——这是 Fable 5 / Opus 4.7/4.8 / Sonnet 5 上大多数编码和 agentic 工作的最佳设置，并且是 Claude Code 的默认值；对大多数需要智能的工作，至少使用 `high`。它在 Fable 5、Opus 4.5、Opus 4.6、Opus 4.7、Opus 4.8、Sonnet 5 和 Sonnet 4.6 上可用；在 Sonnet 4.5 / Haiku 4.5 上会报错。在 Fable 5、Opus 4.7/4.8 和 Sonnet 5 上，effort 相比旧模型更重要——迁移时需要重新调优，并在长周期/agentic 任务上用完整任务规格配合 `high`/`xhigh` 运行。与自适应思考结合时，可以获得最佳的成本-质量权衡。较低 effort 会导致更少、更加聚合的工具调用，更少的前言和更简洁的确认；`high` 通常是兼顾质量和 token 效率的最佳折中；当正确性比成本更重要时使用 `max`；简单任务或 subagents 使用 `low`。

**思考显示 — Fable 5 / Mythos 5 / Opus 4.8 / 4.7 / Sonnet 5 默认是 `"omitted"`：** `display: "summarized"` 会返回一段易读的推理摘要；`"omitted"`（五者默认值，也是从 Opus 4.6 和 Sonnet 4.6 的 `"summarized"` 改成的静默变化）会在流式输出中发送空文本的 `thinking` 块。`display` 只控制可见性——思考仍然发生并计费，和所有设置一样；原始链式思考不会在任何模型上暴露出来。如果要把 reasoning 流式展示给用户，默认看起来像是在输出前长时间停顿——请明确设置 `thinking: {type: "adaptive", display: "summarized"}`。（与 display 无关，当在相同模型上继续对话时，回传 thinking 块时保持原样；其他模型会静默忽略——见迁移指南。）

**Task Budgets（beta，Fable 5 / Opus 4.7 / 4.8 / Sonnet 5）：** `output_config: {task_budget: {type: "tokens", total: N}}` 会告诉模型它在整个 agentic loop 中能够使用多少 token，从而更好地控制节奏并优雅结束，而不是被截断（最小 20,000；beta header `task-budgets-2026-03-13`）。它区别于 `max_tokens`，后者是对单次响应的强制上限，模型并不知道这个上限。见 `shared/model-migration.md` → Task Budgets。

**Sonnet 4.6：** 支持自适应思考（`thinking: {type: "adaptive"}`）。`budget_tokens` 在 Sonnet 4.6 上已弃用——请使用自适应思考。

**较旧模型（仅在用户明确要求时使用）：** 如果用户明确要求 Sonnet 4.5 或其他较旧模型，请使用 `thinking: {type: "enabled", budget_tokens: N}`。`budget_tokens` 必须小于 `max_tokens`（最小 1024）。不要仅仅因为用户提到了 `budget_tokens` 就选择较旧模型——请优先使用带自适应思考的 Opus 4.8。

---

## 压缩（快速参考）

**Beta，Fable 5、Opus 4.8、Opus 4.7、Opus 4.6、Sonnet 5 和 Sonnet 4.6。** 对于可能超过 1M 上下文窗口的长时间对话，启用服务端 compaction。API 会在接近触发阈值时自动总结早期上下文（默认：150K tokens）。需要 beta header `compact-2026-01-12`。

**关键：** 每轮都要把 `response.content`（不是只有文本）追加回 `messages`。compaction block 必须保留——API 会用它在下一次请求中替换压缩后的历史。仅提取文本字符串并追加，会静默丢失 compaction 状态。

见 `{lang}/claude-api/README.md`（Compaction 部分）的代码示例。完整文档可通过 `shared/live-sources.md` 中的 WebFetch 获取。

---

## Prompt Caching（快速参考）

**前缀匹配。** 前缀中任何字节变化都会使其后所有内容失效。渲染顺序是 `tools` → `system` → `messages`。先放稳定内容（冻结 system prompt、确定性的工具列表），把易变内容（时间戳、每次请求 ID、变化的问题）放在最后一个 `cache_control` 断点之后。

**会话中间的 operator instructions**（仅 Claude Opus 4.8；无 beta header）：将 `{"role": "system", ...}` 追加到 `messages[]` 中，而不是修改顶层 `system`。这样可以保留缓存前缀，并且是对 prompt injection 更安全的 operator channel。见 `shared/prompt-caching.md` § Mid-conversation system messages。

**顶层自动缓存**（`cache_control: {type: "ephemeral"}` on `messages.create()`）是最简单的选项，如果你不需要细粒度 placement。每个请求最多 4 个 breakpoint。最小可缓存前缀约 1024 tokens；更短的前缀会静默不缓存。

**通过 `usage.cache_read_input_tokens` 验证**——如果连续多次请求都是 0，说明存在静默 invalidator（如 system prompt 中的 `datetime.now()`、未排序 JSON、变化的 tool set）。

关于 placement pattern、架构指导以及静默 invalidator 审计清单，请阅读 `shared/prompt-caching.md`。语言特定语法见 `{lang}/claude-api/README.md`（Prompt Caching 部分）。

---

## Fast Mode（快速参考）

**研究预览，仅 Opus 4.8 / 4.7。** Opus 4.7 的 fast mode 已弃用——移除后，4.7 上 `speed: "fast"` 会报错。Opus 4.8 是可持续的 fast-capable tier。Fast mode 会用高达 2.5 倍的输出 tokens/秒运行相同模型，但价格更高。每次请求都需要三件事：使用 **beta** messages endpoint（`client.beta.messages.…`）、传入 beta flag `fast-mode-2026-02-01`，并将 `speed: "fast"` 作为顶层请求参数（不是 header，也不是 `extra_body`）。

```python
client.beta.messages.create(
    model="claude-opus-4-8", max_tokens=4096,
    speed="fast", betas=["fast-mode-2026-02-01"],
    messages=[...],
)
```

| 语言 | Beta flag | Speed 参数 |
|---|---|---|
| Python | `betas=["fast-mode-2026-02-01"]` | `speed="fast"` |
| TypeScript / Ruby | `betas: ["fast-mode-2026-02-01"]` | `speed: "fast"` |
| Go | `[]anthropic.AnthropicBeta{anthropic.AnthropicBetaFastMode2026_02_01}` | `Speed: anthropic.BetaMessageNewParamsSpeedFast` |
| Java | `.addBeta(AnthropicBeta.FAST_MODE_2026_02_01)` | `.speed(MessageCreateParams.Speed.FAST)` |
| C# | `Betas = ["fast-mode-2026-02-01"]` | `Speed = Speed.Fast` (`Anthropic.Models.Beta.Messages`) |
| PHP | `betas: ['fast-mode-2026-02-01']` | `speed: 'fast'` |
| cURL | `anthropic-beta: fast-mode-2026-02-01` header | `"speed": "fast"` in body |

`response.usage.speed` 会报告实际使用的速度。Fast mode 有自己单独的速率限制，与标准 Opus 不同；遇到 429 时，要么在 `retry-after` 延迟后重试，要么去掉 `speed` 回退到标准模式（注意：切换 speed 会使 prompt cache 失效）。Fast mode 不适用于 Batch API、Priority Tier、Claude Platform on AWS 或第三方平台。

---

## Task Budgets（快速参考）

**Beta，Fable 5 / Sonnet 5 / Opus 4.8 / 4.7。** Task budget 为 Claude 提供了一个 agentic loop 的 token 上限，让它自己控制节奏并优雅完成，而不是被强行切断。在 `client.beta.messages.stream(...)` 上设置 `output_config` 内的 `task_budget`，并配合 beta flag `task-budgets-2026-03-13`——使用 streaming 是为了避免大 `max_tokens` 触发 HTTP 超时：

```python
with client.beta.messages.stream(
    model="claude-opus-4-8", max_tokens=128000,
    output_config={"effort": "high", "task_budget": {"type": "tokens", "total": 64000}},
    betas=["task-budgets-2026-03-13"],
    messages=[...], tools=[...],
) as stream:
    response = stream.get_final_message()
```

`task_budget` 字段：`type`（始终为 `"tokens"`）、`total`，以及可选 `remaining`（默认等于 `total`）。服务端会注入一个倒计时标记，Claude 会在生成过程中看到它；budget 统计的是 Claude 当前这轮生成的 token 和它读取的工具结果——**不是**你每次请求时重新发回的完整历史。

**观察消耗：** 如果你想展示进度，可以累加 `response.usage.output_tokens`（以及你追加到消息中的工具结果块的 token 数）来计算每轮迭代的开销。正常循环中不要设置 `remaining`——服务端会自己跟踪倒计时；如果同时传了客户端计算的 `remaining`，并且还重新发送完整历史，那么 budget 会被低估。只有当你在两次请求之间压缩或重写历史时，且服务端已经无法推导先前消耗时，才**传入 `remaining`**。

---

## Provider Clients（快速参考）

当目标是第三方平台上的 Claude 时，请使用该平台对应的专用 client 类——不要用带 `base_url` 覆盖的第一方 `Anthropic()` 客户端。构造完成后，这些 client 暴露出的 `messages.create` / `.stream` 表面与第一方 SDK 相同。

### Amazon Bedrock

使用 **Mantle** client（Messages-API Bedrock endpoint）。Bedrock 模型 ID 需要 `anthropic.` 前缀（例如 `"anthropic.claude-opus-4-8"`）。Region 是必填项。

| 语言 | Client |
|---|---|
| Python | `from anthropic import AnthropicBedrockMantle` → `AnthropicBedrockMantle(aws_region="…")` |
| TypeScript | `import { AnthropicBedrockMantle } from "@anthropic-ai/bedrock-sdk"` → `new AnthropicBedrockMantle({ awsRegion: "…" })` |
| Go | `bedrock.NewMantleClient(ctx, bedrock.MantleClientConfig{ AWSRegion: "…" })` |
| Java | `AnthropicOkHttpClient.builder().backend(BedrockMantleBackend.fromEnv()).build()`（来自 `com.anthropic.bedrock.backends`） |
| C# | `new AnthropicBedrockMantleClient(new() { AwsRegion = "…" })`（包 `Anthropic.Bedrock`） |
| PHP | `use Anthropic\Bedrock\MantleClient;` → `new MantleClient(awsRegion: '…')` |
| Ruby | `Anthropic::BedrockMantleClient.new(aws_region: "…")` |

`AnthropicBedrock` / `BedrockClient` / `BedrockBackend`（不带 `Mantle`）是旧版 `bedrock-runtime` InvokeModel 路径——新的代码优先使用 Mantle client。

### Microsoft Foundry

| 语言 | Client |
|---|---|
| Python | `from anthropic import AnthropicFoundry` → `AnthropicFoundry(api_key=…, resource="…")` |
| TypeScript | `import AnthropicFoundry from "@anthropic-ai/foundry-sdk"` → `new AnthropicFoundry({ … })` |
| Java | `AnthropicOkHttpClient.builder().backend(FoundryBackend.fromEnv()).build()`（来自 `com.anthropic.foundry.backends`） |
| C# | `new AnthropicFoundryClient(new AnthropicFoundryApiKeyCredentials(…))`（包 `Anthropic.Foundry`） |
| PHP | `Foundry\Client::withCredentials(…)` |

Go 和 Ruby SDK 目前不支持 Foundry。Ruby 可用标准的 `Anthropic::Client.new(base_url: "<foundry endpoint>")` 作为 fallback（Entra ID 认证没有内置支持）。关于 Claude Platform on AWS，请见 `shared/claude-platform-on-aws.md`。

### Google Cloud Vertex AI

有两个必须传入的构造参数：GCP 的 `project_id` 和 `region`。Vertex 模型 ID 不带前缀——当前生成模型（Opus 4.8/4.7/4.6、Sonnet 5、Sonnet 4.6）使用裸的一方 ID（例如 `"claude-opus-4-8"`）；带日期快照的模型使用 `@` 分隔符（例如 `claude-opus-4-5@20251101`，而不是 `claude-opus-4-5-20251101`）。认证是 GCP ADC（`gcloud auth application-default login`）；没有 Anthropic API key。`region` 可以是 `"global"`（推荐）、多区域（`"us"`/`"eu"`）或具体区域。构造完成后，使用相同的 `messages.create` / `.stream` 表面。

| 语言 | Client |
|---|---|
| Python | `from anthropic import AnthropicVertex` → `AnthropicVertex(project_id="…", region="…")`（安装 `"anthropic[vertex]"`） |
| TypeScript | `import { AnthropicVertex } from "@anthropic-ai/vertex-sdk"` → `new AnthropicVertex({ projectId, region })` |
| Go | `import "github.com/anthropics/anthropic-sdk-go/vertex"` → `anthropic.NewClient(vertex.WithGoogleAuth(ctx, region, projectID))` |
| Java | `AnthropicOkHttpClient.builder().backend(VertexBackend.builder().region("…").project("…").build()).build()`（来自 `com.anthropic.vertex.backends`） |
| C# | `new AnthropicClient { Backend = new VertexBackend(projectId, region) }`（包 `Anthropic.Vertex`） |
| PHP | `use Anthropic\Vertex;` → `Vertex\Client::fromEnvironment(location: '…', projectId: '…')` — 注意这里是 `location` 而不是 `region` |
| Ruby | `Anthropic::VertexClient.new(region: "…", project_id: "…")` |

---

## Context Editing（快速参考）

**Beta。** Context editing 会在模型看到对话前**清除**旧的 tool results 或 thinking blocks；它**不是 compaction**（后者是总结）。在 `client.beta.messages.*` 上配合 beta `context-management-2025-06-27`，传入 `context_management.edits` 和策略类型：

```python
client.beta.messages.create(
    model="claude-opus-4-8", max_tokens=4096,
    betas=["context-management-2025-06-27"],
    context_management={"edits": [{"type": "clear_tool_uses_20250919"}]},
    tools=[...], messages=[...],
)
```

策略类型：`clear_tool_uses_20250919`（清除旧的 tool results；可选 `clear_tool_inputs: true` 也会清除 tool_use 参数）和 `clear_thinking_20251015`（清除 thinking blocks）。不要使用 `compact_20260112` 或 beta `compact-2026-01-12`——它们是独立的 compaction 功能。

---

## 会话中间的 System Messages（快速参考）

**仅 Claude Opus 4.8；无需 beta header。** 将 `{"role": "system", "content": "…"}` 追加到 `messages` 数组中（不是顶层 `system` 字段），以在对话中添加 operator instruction，同时不破坏缓存前缀。使用普通的 `client.messages.create`——没有 beta。会话中间的 system message 必须跟在 `user` 消息后（或者一个以 server-tool use 结尾的 `assistant` 消息后），并且必须是 `messages` 的最后一项，或后面跟着一个 `assistant` 轮次——它不能是 `messages[0]`。可用性见 `shared/platform-availability.md`。见 `shared/prompt-caching.md` § Mid-conversation system messages。

---

## Managed Agents（Beta）

**Managed Agents** 是第三种接入层：由 Anthropic 托管的、带工作区的有状态 agent。你先创建持久化、版本化的 Agent 配置（`POST /v1/agents`），再启动引用它的 Sessions。每个 session 都会为 agent 的工作区提供容器——bash、文件操作和代码执行都会在其中运行；agent 循环本身运行在 Anthropic 的编排层，并通过工具访问容器。session 会流式输出事件；你可以发送消息和工具结果。

可用性：`shared/platform-availability.md`。对于 Bedrock / Vertex / Foundry 上不支持 Managed Agents 的场景，使用 Claude API + tool use。

**强制流程：** Agent（一次）→ Session（每次运行）。`model`/`system`/`tools` 属于 agent，不属于 session。完整阅读指南见 `shared/managed-agents-overview.md`。

**Beta headers：** `managed-agents-2026-04-01`——SDK 会自动为所有 `client.beta.{agents,environments,sessions,vaults,memory_stores,deployments,deployment_runs}.*` 调用设置这个 header。Skills API 使用 `skills-2025-10-02`，Files API 使用 `files-api-2025-04-14`，但除了 `/v1/skills` 和 `/v1/files` 之外，你通常不需要显式传这些 header。

**Subcommands** — 通过 `/claude-api <subcommand>` 直接调用：

| 子命令 | 动作 |
|---|---|
| `managed-agents-onboard` | 从零引导用户搭建一个 Managed Agent。**立即读取 `shared/managed-agents-onboarding.md`**，并按它的访谈脚本执行：**describe → configure the agent（提出建议，不做审问式追问）→ environment → session**（与 Console quickstart 的路径一致，认证推迟到 session 步骤）——默认值和内联建议会完成大部分工作，在任何代码输出前先用静默可行性门控（job vs tools/credentials/data）判断是否适合。不要总结——直接进行访谈。 |

**阅读指南：** 先看 `shared/managed-agents-overview.md`，再看主题性的 `shared/managed-agents-*.md` 文件（core、environments、tools、events、outcomes、multiagent、webhooks、memory、scheduled-deployments、client-patterns、onboarding、api-reference）。对于 Python、TypeScript、Go、Ruby、PHP 和 Java，先读 `{lang}/managed-agents/README.md` 获取代码示例。对于 cURL，读 `curl/managed-agents.md`。**Agents 是持久化的 — 创建一次，引用其 ID。** 请保存 `agents.create` 返回的 agent ID，并将其传给后续的每个 `sessions.create`；不要在请求路径中重复调用 `agents.create`。Anthropic CLI（`ant`）是从版本控制的 YAML 创建 agents 和 environments 的便捷方式——见 `shared/anthropic-cli.md`。如果你需要的 binding 没有出现在语言 README 中，请通过 `shared/live-sources.md` 的相关条目进行 WebFetch，而不要猜测。C# 通过 `client.Beta.Agents` 及相关命名空间提供 beta Managed Agents 支持。

**当用户想从零开始搭建 Managed Agent**（例如“如何开始”“带我一步步创建一个”“搭建一个新的 agent”）时：读取 `shared/managed-agents-onboarding.md` 并执行它的访谈流程——与 `managed-agents-onboard` 子命令相同。

**当用户问“我该如何为 X 编写客户端代码”时：** 使用 `shared/managed-agents-client-patterns.md`——它涵盖了无损流重连、`processed_at` 的 queued/processed 门控、interrupt、`tool_confirmation` 往返、正确的 idle/terminated break 门控、post-idle 状态竞争、stream-first 顺序、file-mount 的陷阱、通过自定义工具将凭据保留在宿主侧等内容。

**当用户希望 agent 按计划运行**（cron、“每天晚上”、“每周报告”）时：读取 `shared/managed-agents-scheduled-deployments.md`——部署会按 cron 节奏自主触发 sessions，并生成每次触发的运行记录和生命周期控制（pause/unpause/archive）。

---

## Server Tools（快速参考）

服务端工具在 Anthropic 的基础设施上运行——不需要客户端执行循环。要在 `tools` 中声明；结果会以内容块的形式返回到同一响应中。**没有 beta header**，除非特别注明。**优先使用你的模型支持的最新类型变体。** 下面的 `_20260209` web search / web fetch 变体（动态过滤）要求 Opus 4.8/4.7/4.6、Sonnet 5 或 Sonnet 4.6；较旧模型的基础变体在表格后列出。

| Tool | `type` | `name` | 关键可选参数 | 结果块类型 |
|---|---|---|---|---|
| Web search | `web_search_20260209` | `web_search` | `max_uses`、`allowed_domains`/`blocked_domains`、`user_location` | `web_search_tool_result` → `.content` 是一组 `web_search_result` |
| Web fetch | `web_fetch_20260209` | `web_fetch` | `max_uses`、`allowed_domains`/`blocked_domains`、`citations`、`max_content_tokens` | `web_fetch_tool_result` → `.content` 是一个带 `document` 块的 `web_fetch_result` |
| Code execution | `code_execution_20260521` | `code_execution` | none | `bash_code_execution_tool_result` → `.content.stdout` / `.stderr` / `.return_code` |
| Tool search (regex) | `tool_search_tool_regex_20251119` | `tool_search_tool_regex` | 将其他工具标记为 `defer_loading: true` | `tool_search_tool_result` |
| Tool search (BM25) | `tool_search_tool_bm25_20251119` | `tool_search_tool_bm25` | 将其他工具标记为 `defer_loading: true` | `tool_search_tool_result` |

`web_search_20260209` / `web_fetch_20260209` 自带动态过滤——代码执行在内部运行，因此**不要**在 `tools` 中另外声明 `code_execution`（第二个执行环境会让模型混乱）。对于比 Opus 4.6 / Sonnet 4.6 更旧的模型，请使用基础变体 `web_search_20250305` / `web_fetch_20250910`；在 Vertex AI 上只有基础 `web_search_20250305` 可用。`code_execution_20260120`（REPL 持久化 + 程序化工具调用）在 Opus 4.5+ / Sonnet 4.5+ 上运行。**Go SDK only**：`code_execution_20260521` 位于 `client.Beta.Messages.New`，需配合 `Betas: []anthropic.AnthropicBeta{"code-execution-2025-08-25"}`（其他语言直接使用普通 `client.messages.create`）；`code_execution_20260120` 在 Go 中和其他地方一样，使用非 beta `client.Messages.New`。Web fetch 只会抓取对话中已经出现过的 URL。不同提供商工具可用性各不相同——见 `shared/platform-availability.md`。关于 `pause_turn` 处理，见 `shared/tool-use-concepts.md`。

## 文档与文件输入（快速参考）

**PDF（base64，无 beta）：** 在用户内容中使用 `{"type": "document", "source": {"type": "base64", "media_type": "application/pdf", "data": <b64 string>}}`，放在文本块之前。Base64 字符串不能包含换行。限制：32 MB 请求，600 页（200k 上下文模型为 100 页）。Java：`ContentBlockParam.ofDocument(DocumentBlockParam... Base64PdfSource.builder().data(...))`。

**Files API（beta `files-api-2025-04-14`）：** 通过 `client.beta.files.upload(...)` 上传 → 响应中的 `id` 即 `file_id`。以 `{"type": "document", "source": {"type": "file", "file_id": "..."}}` 形式引用它来发送 PDF/text，或者 `{"type": "image", ...}` 用于图片——内容块类型必须与文件 MIME type 匹配。上传和引用文件的 `messages.create` 都需要 beta header。可用性：`shared/platform-availability.md`。

**Citations（无 beta）：** 在每个 `document` 内容块上设置 `citations: {enabled: true}`（全部启用或全部不启用）。响应会拆成多个 `text` 块；被引用的块会带有 `citations` 数组。每条 citation 都有 `cited_text`、`document_index`、`document_title`，以及按 `type` 区分的位置：`char_location`（`start_char_index`/`end_char_index`，适用于纯文本）、`page_location`（`start_page_number`/`end_page_number`，1-indexed，适用于 PDF）、`content_block_location`（适用于自定义内容）。与 `output_config.format` 不兼容。

## Tool Use Patterns（快速参考）

**严格工具使用（无 beta）：** 在工具定义上设置 `strict: true` 作为顶层字段（和 `name`/`description`/`input_schema` 并列），而不是放在 `tool_choice` 上。schema 必须包含 `additionalProperties: false` + `required`。这样可以保证 `tool_use.input` 正好按该 schema 校验。Go：`Strict: anthropic.Bool(true)` + 通过 `InputSchema.ExtraFields` 设置 `additionalProperties`；Java：`.strict(true)` + `.putAdditionalProperty("additionalProperties", JsonValue.from(false))`。

**并行工具使用（默认开启）：** 一个 assistant message 可以包含多个 `tool_use` 块。先并发执行它们，然后将所有 `tool_result` 块一次性返回给一个用户消息（不要拆成多个消息）。对于失败工具，返回 `tool_result` 且 `is_error: true`——不要丢弃。

**Tool Runner（SDK beta helper）：** 通过 `client.beta.messages.*` 为你驱动工具调用循环。Python：`@beta_tool` 装饰器 + `client.beta.messages.tool_runner(...)` → `runner.until_done()`。TypeScript：来自 `@anthropic-ai/sdk/helpers/beta/zod` 的 `betaZodTool({...})` + `client.beta.messages.toolRunner(...)` → `await runner`。Go：`toolrunner.NewBetaToolFromJSONSchema(...)` + `client.Beta.Messages.NewToolRunner(...)` → `.RunToCompletion(ctx)`。Java 需要 `.addBeta("structured-outputs-2025-11-13")`。Ruby：`Anthropic::BaseTool` 子类 + `client.beta.messages.tool_runner(...)`。PHP：`BetaRunnableTool` + `->toolRunner(...)`。C#：原始 JSON schema 工具 + `BetaToolRunner` via `client.Beta.Messages.ToolRunner(...)`。

**程序化工具调用（无 beta header）：** Claude 会在代码执行过程中调用你的自定义工具。除了添加 `{"type": "code_execution_20260120", "name": "code_execution"}`，还要在自定义工具上设置 `"allowed_callers": ["code_execution_20260120"]`。适用于 Opus 4.5+ / Sonnet 4.5+（可用性：`shared/platform-availability.md`）。响应待处理的程序化调用时，用户消息必须**只包含** `tool_result` 块（不能有文本）。与 `strict: true`、`disable_parallel_tool_use`、强制 `tool_choice` 或 MCP tools 不兼容。

## Other API Surfaces（快速参考）

**Message Batches（无 beta；可用性：`shared/platform-availability.md`）：** `client.messages.batches.create(requests=[{custom_id, params}, ...])` → 轮询 `client.messages.batches.retrieve(id).processing_status` 直到 `"ended"` → 流式读取 `client.messages.batches.results(id)`。每个结果都有 `.custom_id` + `.result.type`（`succeeded`/`errored`/`canceled`/`expired`）；成功时读取 `.result.message.content`。Python 会把请求包装成 `Request(custom_id=..., params=MessageCreateParamsNonStreaming(...))`。结果以**任意顺序**到达——按 `custom_id` 匹配，而不是按位置。

**Models API（无 beta；可用性：`shared/platform-availability.md`）：** `client.models.list()`（自动分页）和 `client.models.retrieve("claude-opus-4-8")`。每个 model 对象都有 `id`、`display_name`、`created_at`，以及从 2026 年 3 月起新增的 `max_input_tokens`（上下文窗口）、`max_tokens`（输出上限）和 `capabilities`。没有 `context_window` 字段。

**Stop details（GA，Opus 4.7+）：** `response.stop_details` 仅当 `stop_reason == "refusal"` 时才会填充（字段：`type: "refusal"`、`category: "cyber"|"bio"|null`、`explanation`）。对所有其他 `stop_reason`（`end_turn`、`max_tokens`、`tool_use`、`pause_turn` 等）都是 `null`——读取前必须先判断。

**Client config（无 beta）：** `timeout` 默认 10 分钟；**单位因 SDK 而异**——Python/Ruby：秒；TypeScript：**毫秒**；Go `option.WithRequestTimeout(time.Duration)`；Java `Duration`；C# `TimeSpan`。对于非流式请求，TS 会将默认值提升到 60 分钟以适应较大的 `max_tokens`；Java 会在流式请求上这样处理（Java 非流式请求将 30s–10 min 缩放）。`max_retries`/`maxRetries` 默认 2（重试 408/409/429/5xx + 连接错误）。`base_url`（或 `ANTHROPIC_BASE_URL` 环境变量）。每次请求覆盖：Python `client.with_options(timeout=5.0).messages.create(...)`；TS `client.messages.create({...}, {timeout: 5_000})`；Ruby `request_options: {timeout: 5}`。超时会被重试——墙上时间可能达到 `timeout × (max_retries+1)`。

## Workload Identity Federation（快速参考）

**GA，无 beta header。** 构造普通的零参数 client（`Anthropic()` / `new Anthropic()` / `anthropic.NewClient()` / `AnthropicOkHttpClient.fromEnv()`）；当且仅当所有这些变量都设置时，SDK 会自动检测 WIF：`ANTHROPIC_FEDERATION_RULE_ID`、`ANTHROPIC_ORGANIZATION_ID`、`ANTHROPIC_SERVICE_ACCOUNT_ID`、`ANTHROPIC_IDENTITY_TOKEN_FILE`（或 `ANTHROPIC_IDENTITY_TOKEN`），然后在 `/v1/oauth/token` 处交换 JWT 并自动刷新。`ANTHROPIC_WORKSPACE_ID` 不决定激活——只有当 federation rule 跨多个 workspace 时才需要它，否则会返回 400 `workspace_id_required`；对单 workspace rule 是可选项。`ANTHROPIC_API_KEY` 或 `ANTHROPIC_AUTH_TOKEN`（即使为空）会覆盖 WIF，而设置了 `ANTHROPIC_PROFILE` 也会覆盖 federation env vars（缺失的指定 profile 会报错，而不是 fallback）——把这三个都 unset 掉。

---

## 阅读指南

识别好语言后，请根据用户需求读相应文件。

**所有 SDK 语言都使用相同的多文件布局**——目录 `{lang}/claude-api/` 中包含 `README.md`（安装、client init、基本请求、thinking、caching、stop details、misc）、`tool-use.md`（工具定义、agentic loop、Anthropic 定义工具、结构化输出）、`streaming.md`、`batches.md`、`files-api.md`。并不是每种语言都有每个文件（例如 Ruby 没有 `batches.md`）；如果某个文件不存在，说明该功能暂时没有为该语言提供示例——可以退回到 cURL 形态，或者通过 `shared/live-sources.md` WebFetch SDK 仓库。**cURL** → `curl/examples.md`。

下面的“快速任务参考”使用 `{lang}/claude-api/FILE.md` 这种路径记法表示所有语言。

### 快速任务参考

**单次文本分类/摘要/提取/问答：**
→ 只读 `{lang}/claude-api/README.md`

**聊天 UI 或实时响应展示：**
→ 读 `{lang}/claude-api/README.md` + `{lang}/claude-api/streaming.md`

**长时间对话（可能超过上下文窗口）：**
→ 读 `{lang}/claude-api/README.md` — 见 Compaction 部分
**迁移到更新模型（Fable 5 / Opus 4.8 / Opus 4.7 / Opus 4.6 / Sonnet 5 / Sonnet 4.6）或替换已停用模型：**
→ 读 `shared/model-migration.md`
**为 Fable 5 调整提示词或调优（长对话、effort、冗长度、自主运行、sub-agents）：**
→ 读 `shared/model-migration.md` → Migrating to Fable 5 → Behavioral shifts (prompt-tunable) + Long-running agent recommendations
**Prompt caching / 优化缓存 / “为什么我的 cache hit rate 很低”：**
→ 读 `shared/prompt-caching.md` + `{lang}/claude-api/README.md`（Prompt Caching 部分）
**统计文件/提示词/diff 的 token 数（“X 大概有多少 token”）：**
→ 读 `shared/token-counting.md` — 使用 `messages.count_tokens`，不要用 `tiktoken`

**函数调用 / tool use / agents：**
→ 读 `{lang}/claude-api/README.md` + `shared/tool-use-concepts.md` + `{lang}/claude-api/tool-use.md`

**Agent 设计（工具表面、上下文管理、缓存策略）：**
→ 读 `shared/agent-design.md`

**批量处理（非延迟敏感）：**
→ 读 `{lang}/claude-api/README.md` + `{lang}/claude-api/batches.md`

**跨多次请求上传文件：**
→ 读 `{lang}/claude-api/README.md` + `{lang}/claude-api/files-api.md`

**Managed Agents（服务端托管、有状态 agent，带工作区）：**
→ 读 `shared/managed-agents-overview.md` + 其余 `shared/managed-agents-*.md` 文件。对于 Python、TypeScript、Go、Ruby、PHP 和 Java，读 `{lang}/managed-agents/README.md` 获取代码示例。对于 cURL，读 `curl/managed-agents.md`。**Agents 是持久化的 — 创建一次，引用其 ID。** 请保存 `agents.create` 返回的 agent ID，并将其传给后续的每个 `sessions.create`；不要在请求路径中重复调用 `agents.create`。Anthropic CLI（`ant`）是从版本控制的 YAML 创建 agents 和 environments 的便捷方式——见 `shared/anthropic-cli.md`。如果你需要的 binding 没有出现在语言 README 中，请通过 `shared/live-sources.md` 的相关条目进行 WebFetch，而不要猜测。C# 具备 beta Managed Agents 支持——见 `csharp/claude-api/README.md` 或 `curl/managed-agents.md` 中的原始 HTTP 参考。

### Claude API（完整文件参考）

阅读**语言特定的 Claude API 源文件**——每种 SDK 语言对应 `{language}/claude-api/`，cURL 对应 `curl/examples.md`：

1. **`{language}/claude-api/README.md`** — **先读这个。** 包含安装、快速开始、常用模式、错误处理。
2. **`shared/tool-use-concepts.md`** — 当用户需要函数调用、代码执行、memory 或结构化输出时阅读。讲解概念基础。
3. **`shared/agent-design.md`** — 当设计 agent 时阅读：bash 与专用工具、程序化工具调用、tool search/skills、context editing vs compaction vs memory、缓存原则。
4. **`{language}/claude-api/tool-use.md`** — 读语言特定的工具使用代码示例（tool runner、手动 loop、代码执行、memory、结构化输出）。
5. **`{language}/claude-api/streaming.md`** — 在构建聊天 UI 或逐步展示响应的界面时阅读。
6. **`{language}/claude-api/batches.md`** — 当离线处理大量请求时阅读（非延迟敏感）。以 50% 成本异步执行。
7. **`{language}/claude-api/files-api.md`** — 当你要在多次请求中复用同一文件而不重传时阅读。
8. **`shared/prompt-caching.md`** — 当新增或优化 prompt caching 时阅读。涵盖前缀稳定性设计、breakpoint 放置、静默失效的反模式。
9. **`shared/error-codes.md`** — 当调试 HTTP 错误或实现错误处理时阅读。包含每个 SDK 的类型化异常类表与 Go 的 `errors.As` 模式。
10. **`shared/model-migration.md`** — 当升级到新模型、替换已停用模型，或把 `budget_tokens` / prefill 模式迁移到当前 API 时阅读。
11. **`shared/live-sources.md`** — WebFetch 的最新官方文档 URL。

并不是每种语言都有每个文件（例如 Ruby 没有 `batches.md`）；如果某个文件不存在，那说明该功能还没有为该语言提供示例——可以退回到 cURL 形态，或从 `shared/live-sources.md` WebFetch SDK 仓库。

> **说明：** 关于 Managed Agents 的文件参考，请见上面的“## Managed Agents (Beta)”部分——那里列出了所有 `shared/managed-agents-*.md` 文件和语言特定 README。

---

## 何时使用 WebFetch

当满足以下情况时，使用 WebFetch 获取最新文档：

- 用户询问“latest”或“current”信息
- 缓存数据看起来不正确
- 用户询问的功能这里没有覆盖

最新官方文档 URL 在 `shared/live-sources.md` 中。

## 常见坑点

- **没有 `ANTHROPIC_API_KEY` ≠ 没有凭据。** 不要因为环境变量未设置就直接放弃或要求用户提供 key——先运行 `ant auth status`。`ant auth login` 之后，裸 `Anthropic()` 客户端和 `ant …` 都可以在没有环境变量的情况下工作；对原始 curl，请使用 `Authorization: Bearer $(ant auth print-credentials --access-token)` 加上头部 `anthropic-beta: oauth-2025-04-20`。见上面的 Authentication 快速参考和 `shared/anthropic-cli.md`。
- 当把文件或内容传给 API 时，不要静默截断输入。如果内容太长超出上下文窗口，请通知用户并讨论方案（分块、摘要等），而不是悄悄截断。
- **Fable 5 / Sonnet 5 / Opus 4.8 / 4.7 的 thinking：** 仅支持自适应。`thinking: {type: "enabled", budget_tokens: N}` 会返回 400——`budget_tokens` 已经完全移除（连同 `temperature`、`top_p`、`top_k`）。使用 `thinking: {type: "adaptive"}`。Opus 4.8 继承了 4.7 的表面，没有新增 breaking changes；Fable 5 多了一个——显式 `thinking: {type: "disabled"}` 会返回 400（Sonnet 5 / 4.7 / 4.8 会接受）；请改为省略该参数。
- **Opus 4.6 / Sonnet 4.6 的 thinking：** 使用 `thinking: {type: "adaptive"}`——不要为新的 4.6 代码使用 `budget_tokens`（对 Opus 4.6 和 Sonnet 4.6 都已弃用；现有代码的渐进迁移请见 `shared/model-migration.md` 中的 transitional escape hatch——注意这一例外不适用于 Fable 5、Opus 4.7 或 4.8）。对更旧模型来说，`budget_tokens` 必须小于 `max_tokens`（最小 1024）。如果写错会抛错。
- **Prefill 已移除（Fable 5 以及 4.6/4.7/4.8 家族）：** Assistant message 的 prefill（最后一轮 assistant 的预填充）在 Fable 5、Opus 4.6、Opus 4.7、Opus 4.8 和 Sonnet 4.6 上会返回 400。请改用结构化输出（`output_config.format`）或 system prompt 指令控制响应格式。（一个例外：fallback-credit prefill claim——在用 `fallback_has_prefill_claim: true` 赎回信用时，服务端会接受回显的 assistant message；见迁移指南的 refusal 部分。）
- **Fable 5 的 `refusal` stop reason：** 安全分类器可能会拒绝请求——HTTP 200、`stop_reason: "refusal"`（预输出：空 `content`，不计费；流式中途：部分输出已计费——丢弃它）。在读取 `response.content[0]` 前先检查 `stop_reason`，否则会在被拒绝请求上出现 index error。要在另一模型上重试，请按原样回放历史；其他模型会从提示词中丢弃被拒绝模型的 thinking blocks，不计费；无需剥离（而 fallback-credit 赎回必须回显被拒绝的正文，thinking blocks 也要包括）。Fallback 是**可选的**——新的 `claude-fable-5` 代码应默认包含服务端 `fallbacks` 参数，这样拒绝不会让请求直接失败；见上文 Claude Fable 5 部分。
- **Fable 5 tokenizer：** 与 Opus 4.8 使用相同 tokenizer——从 Opus 4.7/4.8 迁移时 token 数量大致不变。从 Opus 4.6、Sonnet、Haiku 或更旧模型迁移时，token 数量不同（Opus 4.7 tokenizer 大约使用 1×–1.35× 更多 token）——请至少用每个模型调用一次 `count_tokens` 并比较 `input_tokens`。
- **在开始编辑前确认迁移范围：** 当用户要求把代码迁移到新的 Claude 模型，但没有指定具体文件、目录或文件列表时，**先询问范围**——整个工作目录、某个子目录，还是某些文件。直到用户确认前不要开始编辑。像“migrate my codebase”“move my project to X”“upgrade to Sonnet 4.6”或裸的“migrate to Opus 4.8”这样的表述仍然有歧义——它们说明了你要做什么，但没有说明要做在哪，所以要先问。只有当提示词明确指明文件、目录或文件列表时，才可以不询问（如“migrate `app.py`”“migrate everything under `services/`”“update `a.py` and `b.py`”）。见 `shared/model-migration.md` 的 Step 0。
- **`max_tokens` 默认值：** 不要把 `max_tokens` 设得太低——命中上限会中途截断输出并迫使重试。对非流式请求，默认使用 `~16000`（保持响应在 SDK HTTP 超时之内）。对流式请求，默认使用 `~64000`（因为超时不是问题，可以给模型更多空间）。只有在有硬性原因时才更低：分类（`~256`）、成本上限、刻意短输出，或者 **`max_tokens: 0`** 用于 cache pre-warming（见 `shared/prompt-caching.md` → Pre-warming）。
- **128K 输出 token：** Fable 5、Opus 4.6、Opus 4.7、Opus 4.8、Sonnet 5 和 Sonnet 4.6 支持高达 128K 的 `max_tokens`，但 SDK 需要使用 streaming 以避免 HTTP 超时。使用 `.stream()` + `.get_final_message()` / `.finalMessage()`。
- **工具调用 JSON 解析（Fable 5 以及 4.6/4.7/4.8 家族）：** Fable 5、Opus 4.6、Opus 4.7、Opus 4.8 和 Sonnet 4.6 生成的工具调用 `input` 字段中的 JSON 字符串转义方式可能不同（例如 Unicode 或正斜杠转义）。请始终用 `json.loads()` / `JSON.parse()` 解析，而不要对序列化后的输入做原始字符串匹配。
- **结构化输出（所有模型）：** 在 `messages.create()` 上使用 `output_config: {format: {...}}`，不要再用已弃用的 `output_format` 参数。这是通用 API 变更，不是 4.6 特有的。
- **不要重新实现 SDK 功能：** SDK 已提供高级帮助函数——请优先使用它们，而不是从头搭建。尤其是：使用 `stream.finalMessage()`，不要手写 `new Promise()` 包装 `.on()` 事件；使用类型化异常类（`Anthropic.RateLimitError` 等），不要用字符串匹配错误文本；使用 SDK 类型（`Anthropic.MessageParam`、`Anthropic.Tool`、`Anthropic.ToolUseBlock` / `Anthropic.ToolResultBlockParam`、`Anthropic.Message` 等），不要重新定义等价接口。
- **错误处理——捕获链式异常，而不是一个大而全的类。** 一个 `except APIStatusError` / `catch (AnthropicServiceException)` / `rescue APIError` 会丢掉可重试错误（429、≥500、网络）和不可重试错误（400/404）之间的区别。请按“最具体优先”的顺序编写链式处理——例如 `NotFoundError` → `RateLimitError` → `APIStatusError` → `APIConnectionError`（或 Go 中的等价写法：先 `errors.As` 到 `*anthropic.Error`，再按 `switch apierr.StatusCode { case 404: …; case 429: …; default: … }`）。语言特定的类名和命名空间在 `shared/error-codes.md` 中。
- **不要先研究 SDK 类型——先写文件。** 如果某个类型名没有在这个技能包含的文档中出现，请先根据语言特定文档中的命名空间/包表写出代码文件，让编译器的报错指向正确的名字。不要把几轮都花在 WebFetch、SDK 仓库 clone 或编译运行一个单独反射程序来发现类型名——先产出源文件，再根据编译器报错修正。对已安装 SDK 做一次 `strings` / `jar tf` / `javap` 查找类型名是可以接受的（几秒内就能出结果），但不要进一步升级。写错类型名的文件是可恢复的；一整轮在没有生成任何文件的情况下做探索则不值得。
- **Bash 和文本编辑工具是 Anthropic 定义的、无 schema 的。** 声明 `{"type": "bash_20250124", "name": "bash"}` / `{"type": "text_editor_20250728", "name": "str_replace_based_edit_tool"}`——没有 `input_schema`。一个名为 `"bash"`、但你自己定义 schema 的自定义工具是另一回事。处理路径和安全检查见 `shared/tool-use-concepts.md` § Client-Side Tools。
- **Advisor tool 模型配对。** advisor tool 的 `model` 必须至少和请求的顶层 `model` 一样强——例如 executor `claude-sonnet-5` → advisor `claude-opus-4-8` 或 `claude-opus-4-7`。不合法的配对会返回 400。配对表见 `shared/tool-use-concepts.md` § Advisor。可用性：`shared/platform-availability.md`。
- **Agent Skills ≠ Managed Agents。** 要让 Claude 通过 Agent Skills 生成 `.pptx`/`.xlsx` 等文件，请调用 `client.beta.messages.create`，配合 `container={"skills": [...]}`、`code_execution_20260521` 工具，以及 `code-execution-2025-08-25` + `skills-2025-10-02` 两个 beta。不要使用 `client.beta.agents` / `sessions` / `environments`——它们是 Managed Agents 表面，而不是 Agent Skills。
- **MCP connector 两端都要有。** `mcp_servers=[{type:"url", url, name}]` 单独出现会被校验拒绝——还要添加 `tools=[{type:"mcp_toolset", mcp_server_name:<same name>}]`，并配合 beta `mcp-client-2025-11-20`。可用性：`shared/platform-availability.md`。
- **Context editing ≠ compaction。** Context editing 会**清除**工具结果和 thinking blocks；compaction 会**总结**历史。对于 context editing，请在 `client.beta.messages.*` 上使用 beta `context-management-2025-06-27`，配合 `context_management.edits` 和类型 `clear_tool_uses_20250919`（或 `clear_thinking_20251015`）——不要使用 `compact_20260112` 类型或 `compact-2026-01-12` beta，那是 compaction。
- **`inference_geo` 是直接顶层请求参数** — `client.messages.create(..., inference_geo="us")` / `.inferenceGeo("us")`。不要把它放进 `extra_body` / `putAdditionalBodyProperty`。适用于 Opus 4.6 / Sonnet 4.6 及之后；可用性：`shared/platform-availability.md`。`response.usage.inference_geo` 会报告运行位置。
- **细粒度工具流式传输不是 beta 功能。** 在工具定义上设置 `eager_input_streaming: true`，然后调用普通的 `client.messages.stream(...)`。没有 beta header 也没有 `client.beta.*` 路径。
- **Cache diagnostics 是 beta。** 使用 `client.beta.messages.*` 和 beta `cache-diagnosis-2026-04-07`。首次请求传 `diagnostics: {previous_message_id: null}`，后续请求传 `diagnostics: {previous_message_id: <previous response id>}`；结果在 `response.diagnostics`。可用性：`shared/platform-availability.md`。
- **Memory tool 类型是 `memory_20250818`。** 声明 `{"type": "memory_20250818", "name": "memory"}`。Go 使用 beta 命名空间类型 `{OfMemoryTool20250818: &anthropic.BetaMemoryTool20250818Param{}}` on `client.Beta.Messages.New`；Python/TypeScript/Ruby/PHP/C# 使用非 beta `client.messages.create`；Java 同时支持非 beta `MemoryTool20250818` 和 beta tool-runner 路径。Python/TypeScript 提供 `BetaAbstractMemoryTool` / `betaMemoryTool` 辅助实现后端。
- **请使用实际支持该功能的模型。** 某些功能仅限于特定模型层级——fast mode 仅 Opus 4.8 / 4.7，task budgets 仅 Fable 5 / Sonnet 5 / Opus 4.8 / 4.7，advisor tool 需要合法的 executor↔advisor 配对。如果用户提示中指定了该功能不支持的模型，请改用支持的模型并在输出中说明替换。
- **Bedrock / Foundry：使用平台客户端类。** Bedrock 使用 `…BedrockMantle…` client（如 Python `AnthropicBedrockMantle`、Java `BedrockMantleBackend`），且模型 ID 需要 `anthropic.` 前缀；不带 `Mantle` 的 `AnthropicBedrock`/`BedrockBackend` 是旧路径。Foundry 使用 `AnthropicFoundry` / `FoundryBackend` / `AnthropicFoundryClient`（C#、Java、PHP、Python、TypeScript）；Go 和 Ruby 没有 Foundry client——Ruby 的文档 fallback 是带自定义 `base_url` 的第一方 client。上面的语言表格已给出。
- **不要为 SDK 数据结构定义自定义类型：** SDK 已导出所有 API 对象的类型。使用 `Anthropic.MessageParam` 表示 messages、`Anthropic.Tool` 表示 tool 定义、`Anthropic.ToolUseBlock` / `Anthropic.ToolResultBlockParam` 表示工具结果、`Anthropic.Message` 表示响应。定义你自己的 `interface ChatMessage { role: string; content: unknown }` 只会重复 SDK 已有内容，并失去类型安全。
- **报告和文档输出：** 对于生成报告、文档或可视化的任务，代码执行沙箱已预装 `python-docx`、`python-pptx`、`matplotlib`、`pillow` 和 `pypdf`。Claude 可以生成格式化文件（DOCX、PDF、图表）并通过 Files API 返回——对于“report”或“document”类请求，可以优先考虑此方案，而不是纯 stdout 文本。
- **服务端工具错误不会抛异常。** Web search 和 web fetch 的错误会返回 HTTP 200，且包含 `web_search_tool_result` / `web_fetch_tool_result` 块，其 `content` 是一个单独的错误对象（如 `{error_code: "max_uses_exceeded"}`）——不是抛出的异常。对于 web search，成功的 `content` 是一个*列表*；错误的 `content` 是一个*对象*——在索引前先分支判断。
- **代码执行输出块类型：** `code_execution_20260521` 返回 `bash_code_execution_tool_result`（带 `.content.stdout`），**不是**旧的裸 `code_execution_tool_result`。请循环 `response.content` 并按正确类型匹配。
- **Tool search：永远不要把所有工具都 defer。** 搜索工具本身不能设置 `defer_loading: true`，并且 `tools` 中至少要有一个非 deferred 工具，否则 API 会返回 400 `All tools have defer_loading set`。
- **`strict: true` 放在 tool 上，而不是 `tool_choice` 上。** 把 `strict` 放在 `tool_choice` 上没有任何效果；它是工具定义本体中和 `name`/`description`/`input_schema` 并列的字段。
- **并行 tool results 要放在一个用户消息中。** 不要把 `tool_result` 块拆到多个用户消息里——这样会静默训练 Claude 停止发并行调用。一个 assistant message 的 `tool_use` 块 → 一个用户 message 的 `tool_result` 块。
- **Citations + structured outputs 不兼容。** 对文档启用 `citations: {enabled: true}` 的同时再设置 `output_config.format` 会返回 400。
- **Batch results 是无序的。** 要按 `custom_id` 匹配，而不是按结果流中的位置。
- **Vertex 模型 ID 没有前缀。** 与 Bedrock 的 `anthropic.` 前缀不同，Vertex 对当前生成模型使用裸的一方 ID（例如 `"claude-opus-4-8"`）；带日期快照模型使用 `@` 分隔（例如 `claude-haiku-4-5@20251001`）。
- **只有在 `stop_reason == "refusal"` 时才会有 `stop_details`。** 对 `max_tokens`、`end_turn` 等，`stop_details` 都是 `null`——读取前要先判断 `.category`。
- **WIF auth：unset `ANTHROPIC_API_KEY`、`ANTHROPIC_AUTH_TOKEN` 和 `ANTHROPIC_PROFILE`。** `ANTHROPIC_API_KEY` 和 `ANTHROPIC_AUTH_TOKEN`（即使设置为 `""`）都会在 SDK 优先级链中压倒 Workload Identity Federation；设置了 `ANTHROPIC_PROFILE` 也会压倒 federation env vars（缺失的指定 profile 会报错，而不是 fallback）。要 `unset` 它们，不要只是置空。
