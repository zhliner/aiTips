# Tool Use 概念

本文件介绍 Claude API 中 tool use 的概念基础。如需各语言的代码示例，请参见 `python/`、`typescript/` 或其他语言文件夹。关于暴露哪些工具的决策启发式、长运行 agent 中的上下文管理以及缓存策略，请参见 `agent-design.md`。

## 用户定义的工具

### 工具定义结构

> **注意：** 使用 Tool Runner（beta）时，工具 schema 会根据你的函数签名（Python）、Zod schema（TypeScript）、注解类（Java）、`jsonschema` struct 标签（Go）或 `BaseTool` 子类（Ruby）自动生成。下方的原始 JSON schema 格式适用于手动方式——包括 PHP 的 `BetaRunnableTool`（它围绕手写 schema 包装一个运行闭包）——或不支持 tool runner 的 SDK。

每个工具需要一个名称、描述和输入的 JSON Schema：

```json
{
  "name": "get_weather",
  "description": "Get current weather for a location",
  "input_schema": {
    "type": "object",
    "properties": {
      "location": {
        "type": "string",
        "description": "City and state, e.g., San Francisco, CA"
      },
      "unit": {
        "type": "string",
        "enum": ["celsius", "fahrenheit"],
        "description": "Temperature unit"
      }
    },
    "required": ["location"]
  }
}
```

**工具定义的最佳实践：**

- 使用清晰、描述性的名称（例如 `get_weather`、`search_database`、`send_email`）
- 编写详细的描述——Claude 使用这些描述来决定何时使用工具。要**规定*何时*调用它**，而不仅仅是它做什么（例如"当用户询问当前价格或近期事件时调用此工具"）。在最近的 Opus 模型上（它们更保守地使用工具），描述中的触发条件可以显著提升should-call 率。
- 为每个属性包含描述
- 对具有固定值集的参数使用 `enum`
- 在 `required` 中标记真正必需的参数；使其他参数可选并提供默认值

---

### Tool Choice 选项

控制 Claude 何时使用工具：

| 值                             | 行为                                      |
| --------------------------------- | --------------------------------------------- |
| `{"type": "auto"}`                | Claude 决定是否使用工具（默认） |
| `{"type": "any"}`                 | Claude 必须使用至少一个工具             |
| `{"type": "tool", "name": "..."}` | Claude 必须使用指定的工具            |
| `{"type": "none"}`                | Claude 不能使用工具                       |

任何 `tool_choice` 值都可以包含 `"disable_parallel_tool_use": true` 以强制 Claude 每次响应最多使用一个工具。默认情况下，Claude 可以在单个响应中请求多个 tool 调用。

---

### Tool Runner 与手动循环

**Tool Runner（推荐）：** SDK 的 tool runner 自动处理 agentic 循环——它调用 API、检测 tool use 请求、执行你的工具函数、将结果反馈给 Claude，并重复直到 Claude 停止调用工具。适用于 Python、TypeScript、Java、Go、Ruby 和 PHP SDK（beta）。Python SDK 还提供 MCP 转换辅助函数（`anthropic.lib.tools.mcp`），用于将 MCP 工具、prompt 和资源转换为 tool runner 可用的格式——详见 `python/claude-api/tool-use.md`。

**手动 Agentic 循环：** 当你需要对循环进行精细控制时使用（例如自定义日志、条件工具执行、人在回路中的审批）。循环直到 `stop_reason == "end_turn"`，始终追加完整的 `response.content` 以保留 tool_use 块，并确保每个 `tool_result` 包含匹配的 `tool_use_id`。

**服务端工具的停止原因：** 使用服务端工具（代码执行、web 搜索等）时，API 运行服务端采样循环。如果此循环达到默认限制 10 次迭代，响应将具有 `stop_reason: "pause_turn"`。要继续，重新发送用户消息和助手响应并发出另一个 API 请求——服务器将从中断处恢复。不要添加额外的用户消息如"继续"——API 检测到尾部的 `server_tool_use` 块并知道自动恢复。

```python
# 在 agentic 循环中处理 pause_turn
if response.stop_reason == "pause_turn":
    messages = [
        {"role": "user", "content": user_query},
        {"role": "assistant", "content": response.content},
    ]
    # 发出另一个 API 请求——服务器自动恢复
    response = client.messages.create(
        model="claude-opus-4-8", messages=messages, tools=tools
    )
```

设置 `max_continuations` 限制（例如 5）以防止无限循环。完整指南请参见：`https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons`

> **安全：** Tool runner 会在 Claude 请求时自动执行你的工具函数。对于具有副作用的工具（发送邮件、修改数据库、金融交易），请在工具函数中验证输入，并考虑对破坏性操作要求确认。如果需要在每次工具执行前进行人在回路中的审批，请使用手动 agentic 循环。

---

### 处理工具结果

当 Claude 使用工具时，响应包含一个 `tool_use` 块。你必须：

1. 使用提供的输入执行工具
2. 在 `tool_result` 消息中发送结果
3. 继续对话

**工具结果中的错误处理：** 当工具执行失败时，设置 `"is_error": true` 并提供信息性错误消息。Claude 通常会确认错误并尝试不同的方法或请求澄清。

**多个工具调用：** Claude 可以在单个响应中请求多个工具。在继续之前处理所有工具——在单个 `user` 消息中发送所有结果。

---

## 服务端工具：代码执行

代码执行工具让 Claude 在安全的沙箱容器中运行代码。与用户定义的工具不同，服务端工具在 Anthropic 的基础设施上运行——你不需要在客户端执行任何操作。只需包含工具定义，Claude 会处理其余部分。

### 关键信息

- 在隔离容器中运行（1 CPU、5 GiB RAM、5 GiB 磁盘）
- 无互联网访问（完全沙箱化）
- Python 3.11，预装数据科学库
- 容器持久化 30 天，可跨请求复用
- 与 web 搜索/web 获取工具一起使用时免费；否则每组织每月 1,550 免费小时后 $0.05/小时

### 工具定义

该工具不需要 schema——只需在 `tools` 数组中声明：

```json
{
  "type": "code_execution_20260120",
  "name": "code_execution"
}
```

Claude 自动获得对 `bash_code_execution`（运行 shell 命令）和 `text_editor_code_execution`（创建/查看/编辑文件）的访问。

### 预装 Python 库

- **数据科学**：pandas、numpy、scipy、scikit-learn、statsmodels
- **可视化**：matplotlib、seaborn
- **文件处理**：openpyxl、xlsxwriter、pillow、pypdf、pdfplumber、python-docx、python-pptx
- **数学**：sympy、mpmath
- **实用工具**：tqdm、python-dateutil、pytz、sqlite3

可在运行时通过 `pip install` 安装额外的包。

### 支持上传的文件类型

| 类型   | 扩展名                         |
| ------ | ---------------------------------- |
| 数据   | CSV、Excel（.xlsx/.xls）、JSON、XML |
| 图像 | JPEG、PNG、GIF、WebP               |
| 文本   | .txt、.md、.py、.js 等          |

### 容器复用

跨请求复用容器以维护状态（文件、已安装包、变量）。从第一个响应中提取 `container_id` 并传递给后续请求。

### 响应结构

响应包含交错的文本和工具结果块：

- `text` — Claude 的解释
- `server_tool_use` — Claude 正在做什么
- `bash_code_execution_tool_result` — 代码执行输出（检查 `return_code` 以确认成功/失败）
- `text_editor_code_execution_tool_result` — 文件操作结果

> **安全：** 在将下载的文件写入磁盘之前，始终使用 `os.path.basename()` / `path.basename()` 清理文件名以防止路径遍历攻击。将文件写入专用输出目录。

---

## 服务端工具：Web 搜索和 Web 获取

Web 搜索和 web 获取让 Claude 搜索互联网并获取页面内容。它们在服务端运行——只需包含工具定义，Claude 会自动处理查询、获取和结果处理。

### 工具定义

```json
[
  { "type": "web_search_20260209", "name": "web_search" },
  { "type": "web_fetch_20260209", "name": "web_fetch" }
]
```

### 动态过滤（Fable 5 / Opus 4.8 / Opus 4.7 / Opus 4.6 / Sonnet 4.6）

`web_search_20260209` 和 `web_fetch_20260209` 版本支持**动态过滤**——Claude 编写并执行代码来过滤搜索结果，然后再将其送入上下文窗口，从而提高准确性和 token 效率。动态过滤内置于这些工具版本中并自动激活；你不需要单独声明 `code_execution` 工具或传递任何 beta header。

```json
{
  "tools": [
    { "type": "web_search_20260209", "name": "web_search" },
    { "type": "web_fetch_20260209", "name": "web_fetch" }
  ]
}
```

不使用动态过滤时，也可以使用之前的 `web_search_20250305` 版本。

> **注意：** 仅当你的应用需要独立于 web 搜索的代码执行功能（数据分析、文件处理、可视化）时，才包含独立的 `code_execution` 工具。将其与 `_20260209` web 工具一起包含会创建第二个执行环境，可能会混淆模型。

---

## 服务端工具：程序化 Tool 调用

在标准 tool use 中，每次工具调用都是一次往返：Claude 调用，结果进入 Claude 的上下文，Claude 推理，然后调用下一个工具。链式调用会累积延迟和 token——大部分中间数据不会再被需要。

程序化 tool 调用让 Claude 将这些调用组合成一个脚本。脚本在代码执行容器中运行；当它调用工具时，容器暂停，调用执行，结果返回到运行中的代码（而非 Claude 的上下文）。脚本使用正常的控制流处理它。只有最终输出返回给 Claude。当需要链接许多工具调用或中间结果较大且应在到达上下文窗口之前进行过滤时使用。

完整文档请使用 WebFetch：

- URL：`https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling`

---

## 服务端工具：Tool 搜索

Tool 搜索工具让 Claude 从大型工具库中动态发现工具，而无需将所有定义加载到上下文窗口中。当你有很多工具但只有少数与任何给定请求相关时使用。发现的工具 schema 被追加到请求中，而非替换——这保持了 prompt 缓存（参见 `agent-design.md` §Caching for Agents）。

完整文档请使用 WebFetch：

- URL：`https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool`

---

## Agent Skills（Messages API）

Agent Skills 将特定任务的指令和文件打包，Claude 在相关时加载它们（例如 Anthropic 预构建的 `pptx`、`xlsx`、`pdf`、`docx` skills）。在 **Messages API** 上，skills 通过 `container` 参数与代码执行工具一起启用——这**不是** Managed Agents 界面，也**不**使用 `client.beta.agents` / `sessions` / `environments`。可用性：参见 `shared/platform-availability.md`。

每个请求必需：

1. `client.beta.messages.create(...)` 同时带有**两个** beta 标志：`code-execution-2025-08-25` **和** `skills-2025-10-02`。
2. `container={"skills": [{"type": "anthropic", "skill_id": "<id>", "version": "latest"}]}` — skills 列表选择在执行容器内可用的 skills。
3. `tools=[{"type": "code_execution_20260521", "name": "code_execution"}]` — skills 通过容器中的代码执行来运行。

```python
response = client.beta.messages.create(
    model="claude-opus-4-8", max_tokens=16000,
    betas=["code-execution-2025-08-25", "skills-2025-10-02"],
    container={"skills": [{"type": "anthropic", "skill_id": "pptx", "version": "latest"}]},
    tools=[{"type": "code_execution_20260521", "name": "code_execution"}],
    messages=[{"role": "user", "content": "Create a 3-slide presentation on X"}],
)
```

生成的文件（`.pptx`、`.xlsx` 等）在容器内写入；响应携带每个文件的文件 ID。通过将该 ID 传递给 Files API 来下载（`client.beta.files.download(file_id)` / `GET /v1/files/{id}/content`，带 `anthropic-beta: files-api-2025-04-14`）。

通过 `GET /v1/skills` 列出可用的 skills（需要 `anthropic-beta: skills-2025-10-02`）。

---

## MCP 连接器（Beta）

MCP 连接器让 Claude 直接从 Messages API 调用托管在远程 MCP 服务器上的工具——Anthropic 在服务端建立 MCP 连接。需要在 `client.beta.messages.create(...)` 上设置 beta 标志 `mcp-client-2025-11-20`。可用性：参见 `shared/platform-availability.md`。

**两个参数需要一起使用：**

- `mcp_servers` — 服务器连接定义数组：`[{"type": "url", "url": "<server URL>", "name": "<server-name>", "authorization_token": "<optional>"}]`
- `tools` — 必须包含一个按名称引用服务器的 `mcp_toolset` 条目：`[{"type": "mcp_toolset", "mcp_server_name": "<server-name>"}]`

toolset 中的 `mcp_server_name` 必须匹配 `mcp_servers` 中的一个 `name`。省略 `mcp_toolset` 条目会被拒绝为验证错误——`mcp_servers` 中的每个服务器都必须被恰好一个 toolset 引用。

```python
client.beta.messages.create(
    model="claude-opus-4-8", max_tokens=1024,
    betas=["mcp-client-2025-11-20"],
    mcp_servers=[{"type": "url", "url": "https://example/sse", "name": "example-mcp"}],
    tools=[{"type": "mcp_toolset", "mcp_server_name": "example-mcp"}],
    messages=[...],
)
```

Go 使用类型化常量 `anthropic.AnthropicBetaMCPClient2025_11_20`；旧的 `…2025_04_04` 常量已弃用。

可选的 toolset 字段：`default_config`（所有工具的默认值，例如 `{"enabled": false}` 用于白名单模式）和 `configs`（按工具名称键控的每工具覆盖）。

---

## Tool Use 示例

你可以在工具定义中直接提供示例工具调用，以展示使用模式并减少参数错误。这有助于 Claude 理解如何正确格式化工具输入，特别是对于具有复杂 schema 的工具。

完整文档请使用 WebFetch：

- URL：`https://platform.claude.com/docs/en/agents-and-tools/tool-use/implement-tool-use`

---

## 客户端工具：Computer Use

Computer use 让 Claude 与桌面环境交互（截图、鼠标、键盘）。它是一个客户端工具——你的应用提供环境并执行 Claude 请求的操作；Anthropic 实时处理截图和操作请求，但不托管环境或保留数据。

完整文档请使用 WebFetch：

- URL：`https://platform.claude.com/docs/en/agents-and-tools/computer-use/overview`

---

## 上下文编辑

上下文编辑在长运行 agent 累积轮次时，清除记录中过时的工具结果和 thinking 块。与压缩（总结）不同，上下文编辑是修剪——被清除的内容被移除而非替换。当旧的工具输出不再相关且你希望保持记录精简而不丢失对话结构时使用。

**Beta。** 使用 `client.beta.messages.*` 并带 beta `context-management-2025-06-27`。通过 `context_management.edits` 配置，策略类型为 `clear_tool_uses_20250919`（清除旧的工具结果；可选 `clear_tool_inputs: true` 同时清除 tool_use 参数）或 `clear_thinking_20251015`（清除 thinking 块）。这些**不是**压缩类型——`compact_20260112` 带 beta `compact-2026-01-12` 是单独的压缩功能。

完整文档请使用 WebFetch：

- URL：`https://platform.claude.com/docs/en/build-with-claude/context-editing`

---

## 服务端工具：Advisor（Beta）

Advisor 工具将更快的、更低成本的**执行者**模型（请求中的顶层 `model`）与更高智能的**顾问**模型（工具定义中的 `model` 字段）配对，顾问在生成过程中提供战略指导。执行者完成大部分 token 生成；顾问在规划时被咨询。可用性：参见 `shared/platform-availability.md`。

### 工具定义

```json
{
  "type": "advisor_20260301",
  "name": "advisor",
  "model": "claude-opus-4-8"
}
```

**顾问模型必须至少与执行者一样强大。** 无效的配对会返回 `400 invalid_request_error`。有效配对：

| 执行者（请求 `model`） | 有效顾问（工具 `model`） |
|---|---|
| `claude-haiku-4-5` / `claude-sonnet-4-6` / `claude-sonnet-5` / `claude-opus-4-6` / `claude-opus-4-7` | `claude-opus-4-8` 或 `claude-opus-4-7` |
| `claude-opus-4-8` | 仅 `claude-opus-4-8` |

通过 `client.beta.messages.create(...)` 调用，带 `betas=["advisor-tool-2026-03-01"]`（或 `anthropic-beta: advisor-tool-2026-03-01` header）。在多轮对话中，将完整的 `response.content`——包括任何 `advisor_tool_result` 块——追加回下一轮的 `messages`。如果你在后续轮次中从 `tools` 中移除 advisor 工具，而历史记录仍包含 `advisor_tool_result` 块，API 会返回 400。

---

## 客户端工具：Memory

Memory 工具使 Claude 能够通过记忆文件目录跨对话存储和检索信息。Claude 可以创建、读取、更新和删除在会话之间持久化的文件。

### 关键信息

- 客户端工具——你通过实现控制存储
- 支持命令：`view`、`create`、`str_replace`、`insert`、`delete`、`rename`
- 操作 `/memories` 目录中的文件
- Python、TypeScript 和 Java SDK 提供辅助类/函数来实现 memory 后端

> **安全：** 永远不要在记忆文件中存储 API 密钥、密码、token 或其他秘密。对 personally identifiable information（PII）保持谨慎——在持久化用户数据之前检查数据隐私法规（GDPR、CCPA）。参考实现没有内置访问控制；在多用户系统中，在你的工具处理器中实现按用户的记忆目录和身份验证。

完整实现示例请使用 WebFetch：

- 文档：`https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool.md`

---

## 客户端工具：Bash 和文本编辑器

Bash 和文本编辑器工具是 **Anthropic 定义的、无 schema** 的工具。仅通过 `type` 和 `name` 声明——输入 schema 内置于模型中，无法修改。**不要传递 `input_schema`**，也不要定义恰好名为 `"bash"` 的自定义工具——那会创建一个没有内置行为的用户定义工具。

两者都是**客户端执行的**：Claude 返回一个 `tool_use` 块，你的代码在本地执行操作，然后你发送回一个 `tool_result`。API 是无状态的；你的应用在轮次之间维护 shell 会话或文件系统。

### Bash 工具声明

```json
{"type": "bash_20250124", "name": "bash"}
```

| 语言 | 声明 |
|---|---|
| Python / TypeScript / Ruby / cURL | 普通对象 `{"type": "bash_20250124", "name": "bash"}` |
| Go | `anthropic.ToolUnionParam{OfBashTool20250124: &anthropic.ToolBash20250124Param{}}` |
| Java | `.addTool(ToolBash20250124.builder().build())` 来自 `com.anthropic.models.messages` |
| C# | `Tools = [new ToolBash20250124()]` 来自 `Anthropic.Models.Messages` |
| PHP | `tools: [new \Anthropic\Messages\ToolBash20250124()]` |

Claude 的 `tool_use.input` 包含 `{"command": "<string>"}` 或 `{"restart": true}`。先检查 `restart`（重置会话，返回确认字符串）；否则运行 `command` 并返回合并的 stdout + stderr。

> **安全——命令是不可信的模型输出。** 在隔离环境中运行（容器、VM 或受限用户）；应用允许执行的可执行文件的**白名单**并拒绝 shell 操作符（`&&`、`|`、`;`、`` ` ``、`$()`）；设置超时和资源限制；记录每个命令。黑名单不够充分。

### 文本编辑器工具声明

```json
{"type": "text_editor_20250728", "name": "str_replace_based_edit_tool"}
```

可选字段：`max_characters` 用于限制 `view` 输出。Java 提供类型化的 `ToolTextEditor20250728` 构建器（`com.anthropic.models.messages`）；其他静态类型 SDK 遵循相同的命名模式——确切类请参见 `{lang}/claude-api/tool-use.md` 中的 Anthropic-Defined Tools 章节。

> **安全——`path` 是不可信的模型输出。将所有文件操作限制在固定的项目根目录内。** 在执行任何命令之前，将模型提供的 `path` 解析为其规范形式并验证它仍在项目根目录内；如果它逃逸则拒绝请求（`..`、符号链接、根目录外的绝对路径、URL 编码的遍历如 `%2e%2e%2f`）。使用你的语言的内置路径工具（例如 Python 的 `pathlib.Path.resolve()` 然后检查 `.is_relative_to(root)`）。永远不要直接对原始 `path` 值调用 `open()` / `writeFile` / `unlink`。

`tool_use.input.command` 为以下之一：

| `command` | 其他输入 | 操作 |
|---|---|---|
| `view` | `path`，可选 `view_range` | 返回文件内容或目录列表 |
| `create` | `path`、`file_text` | 使用 `file_text` 创建/覆盖文件。如果文件已存在则创建备份。 |
| `str_replace` | `path`、`old_str`、`new_str` | 替换恰好一个匹配项；0 或 >1 个匹配时报错 |
| `insert` | `path`、`insert_line`、`insert_text` | 在第 `insert_line` 行之后插入 `insert_text`（0 = 文件开头） |

对于两个工具，出错时返回 `{"type": "tool_result", "tool_use_id": "…", "content": "<error text>", "is_error": true}` 以便 Claude 可以恢复。

---

## 结构化输出

结构化输出约束 Claude 的响应遵循特定的 JSON schema，保证有效、可解析的输出。这不是一个单独的工具——它增强了 Messages API 响应格式和/或工具参数验证。

提供两个功能：

- **JSON 输出**（`output_config.format`）：控制 Claude 的响应格式
- **严格 tool use**（`strict: true`）：保证有效的工具参数 schema

**支持的模型：** Claude Fable 5、Claude Opus 4.8、Claude Sonnet 5 和 Claude Haiku 4.5。旧版模型（Claude Opus 4.5、Claude Opus 4.1）也支持结构化输出。

> **推荐：** 使用 `client.messages.parse()` 自动根据 schema 验证响应。直接使用 `messages.create()` 时，使用 `output_config: {format: {...}}`。`output_format` 便捷参数也被某些 SDK 方法接受（例如 `.parse()`），但 `output_config.format` 是规范的 API 级参数。

### JSON Schema 限制

**支持：**

- 基本类型：object、array、string、integer、number、boolean、null
- `enum`、`const`、`anyOf`、`allOf`、`$ref`/`$def`
- 字符串格式：`date-time`、`time`、`date`、`duration`、`email`、`hostname`、`uri`、`ipv4`、`ipv6`、`uuid`
- `additionalProperties: false`（所有对象必需）

**不支持：**

- 递归 schema
- 数值约束（`minimum`、`maximum`、`multipleOf`）
- 字符串约束（`minLength`、`maxLength`）
- 复杂的数组约束
- `additionalProperties` 设置为 `false` 以外的任何值

Python 和 TypeScript SDK 通过从发送到 API 的 schema 中移除不支持的约束并在客户端验证它们来自动处理。

### 重要说明

- **首次请求延迟**：新 schema 会产生一次性编译成本。后续使用相同 schema 的请求使用 24 小时缓存。
- **拒绝**：如果 Claude 因安全原因拒绝（`stop_reason: "refusal"`），输出可能不匹配你的 schema。
- **Token 限制**：如果 `stop_reason: "max_tokens"`，输出可能不完整。增加 `max_tokens`。
- **不兼容**：引用（返回 400 错误）、消息预填充。
- **兼容**：Batches API、流式传输、token 计数、extended thinking。

---

## 有效使用工具的技巧

1. **提供详细描述**：Claude 严重依赖描述来理解何时以及如何使用工具
2. **使用具体的工具名称**：`get_current_weather` 比 `weather` 更好
3. **验证输入**：始终在执行前验证工具输入
4. **优雅处理错误**：返回信息性错误消息以便 Claude 可以适应
5. **限制工具数量**：过多工具可能会混淆模型——保持工具集聚焦
6. **测试工具交互**：验证 Claude 在各种场景中正确使用工具

详细 tool use 文档请使用 WebFetch：

- URL：`https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview`