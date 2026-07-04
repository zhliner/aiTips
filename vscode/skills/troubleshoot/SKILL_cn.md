---
name: troubleshoot
description: 通过分析 JSONL 文件中的直接调试日志来调查 chat agent 的异常行为。当用户询问为什么发生了某事、为什么请求很慢、为什么使用或跳过了某些工具或 subagent，或者为什么 instructions/skills/agents 没有加载时使用。
---

# 故障排查

## 用途

本 skill 使用直接日志文件来调查和解释 chat agent 的异常行为。

适用于以下问题：
- 为什么这个请求耗时这么久？
- 为什么调用了某个工具或 subagent？
- 为什么 instruction/skill/agent 文件没有加载？
- 为什么工具调用被阻止或失败了？
- 为什么模型没有按预期执行？

结论必须基于日志中的证据。不要猜测。

## 数据来源

- 待分析的目标 session 日志目录：`{{VSCODE_TARGET_SESSION_LOG}}`

使用 Copilot Chat 写入的直接调试日志文件：

```
debug-logs/<sessionId>/
  main.jsonl                              — 始终从这里开始；主要对话日志
  models.json                             — （可选）session 开始时可用模型的快照
  system_prompt_0.json                    — （可选）发送给模型的完整 system prompt（未截断）
  system_prompt_1.json                    — （可选）当 session 中途切换模型时写入
  tools_0.json                            — （可选）发送给模型的工具定义
  tools_1.json                            — （可选）当 session 中途切换模型时写入
  runSubagent-<agentName>-<uuid>.jsonl    — （可选）subagent 的工具调用和 LLM 请求
  searchSubagent-<uuid>.jsonl             — （可选）搜索 subagent 的工作
  title-<uuid>.jsonl                      — （可选，仅 UI）标题生成
  categorization-<uuid>.jsonl             — （可选，仅 UI）提示分类
  summarize-<uuid>.jsonl                  — （可选，仅 UI）对话摘要
```

始终先读取 `main.jsonl` — 它包含完整的对话流程。子文件仅在对应操作发生时才会出现。`main.jsonl` 包含 `child_session_ref` 条目，通过文件名链接到每个子文件。Title、categorization 和 summarize 文件是 UI 管理文件，与故障排查关系不大。当调查模型可用性或选择问题时，读取 `models.json` — 它包含 session 开始时可用的完整模型列表（包括能力、计费和限制信息）。

当调查模型被告知的内容（system prompt、instructions）时，读取 `main.jsonl` 中 `system_prompt_ref` 条目引用的 `system_prompt_*.json` 文件。该文件包含完整的未截断 system prompt，格式为 `{ "content": "..." }`。当调查哪些工具可用时，同样读取对应的 `tools_*.json` 文件。如果 session 中途模型发生了变更，则存在多个编号文件 — 每个 `llm_request` 条目都有一个 `systemPromptFile` 属性，指示该请求使用的是哪个文件。

每行是一个 JSON 对象。常见字段：`ts`（epoch 毫秒）、`dur`（持续时间毫秒）、`sid`（session ID）、`type`、`name`、`spanId`、`parentSpanId`、`status`（`ok`|`error`）、`attrs`（类型特定的详细信息）。

### 事件类型参考及示例

#### discovery — 自定义文件加载（instructions、skills、agents、hooks）
```jsonl
{"ts":1773200251309,"dur":0,"sid":"62f52dec","type":"discovery","name":"Load Instructions","spanId":"2cb1f2f4","status":"ok","attrs":{"details":"Resolved 0 instructions in 0.0ms | folders: [/c:/Users/user/.copilot/instructions, /workspace/.github/instructions]","category":"discovery","source":"core"}}
{"ts":1773200251415,"dur":0,"sid":"62f52dec","type":"discovery","name":"Load Agents","spanId":"38a897d8","status":"ok","attrs":{"details":"Resolved 3 agents in 0.0ms | loaded: [Plan, Ask, Explore] | folders: [/workspace/.github/agents]","category":"discovery","source":"core"}}
{"ts":1773200251431,"dur":0,"sid":"62f52dec","type":"discovery","name":"Load Skills","spanId":"472eb225","status":"ok","attrs":{"details":"Resolved 6 skills in 0.0ms | loaded: [agent-customization, troubleshoot, ...]","category":"discovery","source":"core"}}
```
关键 attrs：`details`（人类可读的摘要，包含文件夹路径、已加载项目、跳过原因）、`category`（始终为 `"discovery"`）、`source`（`"core"`）。

#### tool_call — 工具调用（成功或失败）
```jsonl
{"ts":1773200222647,"dur":4,"sid":"62f52dec","type":"tool_call","name":"manage_todo_list","spanId":"000000000000000b","parentSpanId":"0000000000000003","status":"ok","attrs":{"args":"{\"operation\":\"read\"}","result":"No todo list found."}}
{"ts":1773200234047,"dur":8937,"sid":"62f52dec","type":"tool_call","name":"run_in_terminal","spanId":"000000000000000d","parentSpanId":"0000000000000003","status":"error","attrs":{"args":"{\"command\":\"echo rama\"}","result":"ERROR: conpty.node missing","error":"A native exception occurred during launch"}}
```
关键 attrs：`args`（工具输入的 JSON 字符串）、`result`（工具输出或错误文本）、`error`（当 `status:"error"` 时存在）。

#### llm_request — 模型往返请求
```jsonl
{"ts":1773200231010,"dur":3001,"sid":"62f52dec","type":"llm_request","name":"chat:gpt-4o","spanId":"000000000000000c","parentSpanId":"0000000000000003","status":"ok","attrs":{"model":"gpt-4o","inputTokens":15025,"outputTokens":126,"ttft":1987,"maxTokens":32000,"systemPromptFile":"system_prompt_0.json","userRequest":"echo hello","inputMessages":"[{...}]"}}
```
关键 attrs：`model`、`inputTokens`、`outputTokens`、`ttft`（首 token 时间，毫秒）、`maxTokens`、`temperature`、`topP`、`systemPromptFile`（引用 session 目录中的 system prompt 文件）、`toolsFile`（引用 session 目录中的工具文件）、`userRequest`（完整的用户消息内容，未截断）、`inputMessages`（完整的消息数组，JSON 格式，截断至配置的 `maxAttributeSizeChars` — 默认无限制）、`error`（失败时存在）。

#### agent_response — 模型输出（文本 + 工具调用）
```jsonl
{"ts":1773200234011,"dur":0,"sid":"62f52dec","type":"agent_response","name":"agent_response","spanId":"agent-msg-000000000000000c","parentSpanId":"0000000000000003","status":"ok","attrs":{"response":"[{\"role\":\"assistant\",...}]","reasoning":"The user wants me to run a command."}}
```
关键 attrs：`response`（JSON 编码的消息部分数组；可能被截断）、`reasoning`（可选 — 当 thinking 模式激活时，模型的思维链/思考文本；可能被截断）。

#### user_message — 用户输入
```jsonl
{"ts":1773200251345,"dur":0,"sid":"62f52dec","type":"user_message","name":"user_message","spanId":"000000000000000f","status":"ok","attrs":{"content":"using subagent count .md"}}
```
关键 attrs：`content`（用户的消息文本）。

#### subagent — subagent 调用
```jsonl
{"ts":1773200254954,"dur":7921,"sid":"62f52dec","type":"subagent","name":"Explore","spanId":"0000000000000014","parentSpanId":"0000000000000013","status":"ok","attrs":{"agentName":"Explore"}}
```
关键 attrs：`agentName`、`description`（可选）、`error`（失败时存在）。

#### generic — 杂项事件
```jsonl
{"ts":1773200260000,"dur":0,"sid":"62f52dec","type":"generic","name":"some-event","spanId":"abc123","status":"ok","attrs":{"details":"Additional context","category":"some-category"}}
```

特殊 generic 条目：
- `system_prompt_ref` — 引用 session 目录中的 `system_prompt_*.json` 文件。`attrs.file` 为文件名，`attrs.model` 为其写入的目标模型。读取此文件可查看完整的 system prompt。
- `tools_ref` — 引用 `tools_*.json` 文件。`attrs.file` 为文件名，`attrs.model` 为目标模型。

#### session_start — session 元数据（在 session 开始时出现一次）
```jsonl
{"ts":1773200251300,"dur":0,"sid":"62f52dec","type":"session_start","name":"session_start","spanId":"session-start-62f52dec","status":"ok","attrs":{"copilotVersion":"0.43.2026033104","vscodeVersion":"1.99.0"}}
```
关键 attrs：`copilotVersion`、`vscodeVersion`。用于识别生成日志的构建版本。

#### turn_start / turn_end — 工具调用循环迭代边界
```jsonl
{"ts":1773200251400,"dur":0,"sid":"62f52dec","type":"turn_start","name":"turn_start:0","spanId":"turn-start-X-0","status":"ok","attrs":{"turnId":"0"}}
{"ts":1773200255000,"dur":0,"sid":"62f52dec","type":"turn_end","name":"turn_end:0","spanId":"turn-end-X-0","status":"ok","attrs":{"turnId":"0"}}
```
关键 attrs：`turnId`（单次用户请求的工具调用循环中的迭代编号）。用于识别事件属于哪次迭代，以及统计总循环迭代次数。

### 读取事件层级

事件通过 `spanId`/`parentSpanId` 形成树状结构。典型链路：
1. `user_message`（spanId：`X`）— 用户的回合
2. `llm_request`（parentSpanId：`X`）— 该回合的模型调用
3. `agent_response`（parentSpanId：`X`）— 模型返回的内容
4. `tool_call`（parentSpanId：`X`）— 从响应中执行的工具
5. 另一个 `llm_request`（parentSpanId：`X`）— 工具结果后的下一次模型调用

Subagent 调用会创建嵌套层级：`runSubagent` 的 `tool_call`（spanId：`Y`）成为子 `subagent` span 的父节点，而该子 span 又成为其自身 `llm_request`/`tool_call` 事件的父节点。

## 工具策略（重要）

调试日志文件位于 workspace 之外（在用户存储目录中），因此 workspace 范围的搜索工具如 `grep_search` 无法访问它们。请改用终端。

**不要对日志文件使用 `grep_search` — 它仅适用于 workspace 文件。**

### macOS / Linux / WSL / Git Bash

使用 `run_in_terminal` 配合 `grep` 或 `jq`：
- 查找错误：`grep '"status":"error"' <logPath>`
- 查找 discovery 事件：`grep '"type":"discovery"' <logPath>`
- 查找慢事件（持续时间 > 5 秒）：`jq -c 'select(.dur > 5000)' <logPath>`
- 查找工具调用：`grep '"type":"tool_call"' <logPath>`
- 搜索特定文本：`grep 'search_term' <logPath>`
- 获取最后 N 行：`tail -n 50 <logPath>`
- 按类型统计事件数量：`jq -r '.type' <logPath> | sort | uniq -c | sort -rn`
- 提取特定字段：`jq -c '{type, name, status, dur}' <logPath>`
- 按类型过滤并显示详情：`jq -c 'select(.type == "discovery")' <logPath>`
- 查找用户消息：`jq -c 'select(.type == "user_message") | .attrs.content' <logPath>`

### Windows（PowerShell）

使用 `run_in_terminal` 配合 PowerShell 命令：
- 查找错误：`Select-String '"status":"error"' <logPath>`
- 查找 discovery 事件：`Select-String '"type":"discovery"' <logPath>`
- 查找工具调用：`Select-String '"type":"tool_call"' <logPath>`
- 搜索特定文本：`Select-String 'search_term' <logPath>`
- 获取最后 N 行：`Get-Content <logPath> -Tail 50`
- 使用 Node.js 解析和过滤（始终可用）：`node -e "require('fs').readFileSync('<logPath>','utf8').split('\n').filter(Boolean).map(JSON.parse).filter(e => e.dur > 5000).forEach(e => console.log(JSON.stringify(e)))"`
- 按类型统计事件数量：`node -e "const lines=require('fs').readFileSync('<logPath>','utf8').split('\n').filter(Boolean).map(JSON.parse);const c={};lines.forEach(e=>c[e.type]=(c[e.type]||0)+1);Object.entries(c).sort((a,b)=>b[1]-a[1]).forEach(([t,n])=>console.log(n,t))"`

### 通用规则

- **日志文件可能非常大**（长时间 session 可达数十 MB 或更多）。如果不确定，始终先检查文件大小：`ls -lh <logPath>`（Windows 上使用 `(Get-Item <logPath>).Length`）。如果文件很大，避免使用将整个文件加载到内存中的命令（如使用 `readFileSync` 的 `node -e`）。优先使用流式工具如 `grep`、`jq`、`Select-String`、`tail` 或 `head`。
- 仅在已知行号后，对小的目标范围使用 `read_file`（几行）。永远不要读取整个日志文件。
- 使用 `run_in_terminal` 配合 `ls -lh`（Windows 上使用 `dir`）来定位候选 `.jsonl` 文件并检查其大小。
- 在 Windows 上，如果 `grep`/`jq` 不可用，退回到使用 `Select-String` 或 `node -e` 单行命令（仅适用于较小的文件）。

## 调查工作流

1. 确定日志文件
- **聚焦于上方 Runtime Log Context 部分提供的 session 日志目录。** 首先读取每个目录中的 `main.jsonl`。
- 如果列出了多个 session 目录，调查所有目录 — 用户希望跨 session 进行比较或查找共同模式。
- 仅当问题跨越多个 session 或提供的 session 不包含相关事件时，才搜索 debug-logs 目录中的其他 session 文件。

2. 通过 `run_in_terminal` 快速分诊（macOS/Linux 使用 `grep`/`jq`，Windows 使用 `Select-String`/`node -e`）
- 错误：搜索 `"status":"error"`
- 延迟：过滤高 `dur` 值（> 5000）
- Discovery 问题：搜索 `"type":"discovery"`
- 工具行为：搜索 `"type":"tool_call"`
- 模型行为：搜索 `"type":"llm_request"`

3. 仅读取相关片段
- 提取可疑事件周围的精确行。
- 必要时通过 `spanId` / `parentSpanId` 进行关联。

4. 确定根本原因
- 从证据中选择最可能的原因。
- 如果有多个因素，按影响程度排序。

5. 提供修复建议
- 尽可能提供具体的后续步骤。

## 网络问题调查

如果怀疑存在网络连接或认证问题（例如，日志中反复出现请求超时、401/403 错误或模型端点故障），使用 `run_vscode_command` 工具运行 VS Code 命令 `github.copilot.debug.collectDiagnostics`。该命令以字符串形式返回完整的诊断报告，可以直接从工具输出中读取结果。报告包含：
- 认证和 token 状态
- 网络可达性检查
- 代理和证书配置
- 扩展和环境详情

该命令还会在编辑器中打开报告供用户查看。使用返回的字符串来诊断网络问题。

## 自定义文档参考

当调查与特定类型的自定义文件（instructions、prompt 文件、agents 等）相关的问题，且需要有关预期格式或行为的更多详细信息时，加载相关文档页面：

- Custom instructions：`https://code.visualstudio.com/docs/copilot/customization/custom-instructions`
- Prompt files：`https://code.visualstudio.com/docs/copilot/customization/prompt-files`
- Custom agents：`https://code.visualstudio.com/docs/copilot/customization/custom-agents`
- Language models：`https://code.visualstudio.com/docs/copilot/customization/language-models`
- MCP servers：`https://code.visualstudio.com/docs/copilot/customization/mcp-servers`
- Hooks：`https://code.visualstudio.com/docs/copilot/customization/hooks`
- Agent plugins：`https://code.visualstudio.com/docs/copilot/customization/agent-plugins`

当需要验证文件格式预期、确认支持的字段，或帮助用户修复自定义文件时使用这些文档。

## 最后手段 — Copilot Issues Wiki

当调查未得出明确的根本原因或没有具体的修复建议时：

1. 加载 Copilot Issues wiki 页面：`https://github.com/microsoft/vscode/wiki/Copilot-Issues`。
2. 在返回的 wiki 内容中搜索与用户问题相关的部分。
3. 在回复中总结 wiki 中适用的故障排查步骤。
4. 如果 wiki 包含相关指导，将这些步骤作为具体建议呈现给用户。
5. 如果 wiki 也没有相关信息，告知用户："诊断日志未显示此行为的明确原因，已知问题 wiki 也未涵盖此场景。请考虑在 https://github.com/microsoft/vscode/issues 提交 issue。"

## 回复指南

回复应涵盖：
- 发生了什么以及原因（根本原因或最可能的解释）
- 日志中的关键证据（意译，而非原始转储）
- 如何修复或下一步尝试什么

不需要为每一项单独设置标题。对于简单的问题，一个简短的综合说明加上"How to fix"部分即可。仅在问题复杂或涉及多个因素时使用更多结构（标题、多个部分）。

### 格式
- 使用标题、项目符号和粗体使回复易于快速浏览。
- 保持段落简短。优先使用列表而非大段文字。
- 引用日志证据时，意译或总结而非粘贴原始日志行。如果某个具体细节很重要（如错误消息），仅引用该消息 — 而非整条日志条目。
- 当比较多个值或事件时使用表格（如名称不匹配、各步骤的延迟分解）。

### 抽象层次
- 不要叙述调查过程。永远不要说"我正在调查 session 调试日志…"、"我在日志中发现了关键线索…"或"我正在检查 skill 文件元数据…"。直接跳到发现。
- 不要使用内部术语如"discovery summary"、"frontmatter name"、"the loader"、"event hierarchy"或"span tree"。用用户能够据此行动的通俗语言描述发生了什么。
- 不要向用户描述内部日志文件结构、事件类型或 JSONL 格式 — 他们不需要知道这些实现细节。
- 抽象地引用日志文件（如"调试日志"或"session 日志"），而非使用字面文件名如 `main.jsonl` 或 `runSubagent-Explore-abc123.jsonl`。
- 关注发生了什么以及为什么，而非你是如何发现的。

### 示例

**错误**（叙述调查过程，使用内部术语，粘贴原始日志）：
> 我正在调查 session 调试日志以确认 testing skill 是否被发现。在 skills discovery summary 中，loader 报告：`skipped: testing2 (name-mismatch)`。SKILL.md 中的 frontmatter name 与文件夹标识不匹配。

**正确**（简洁、用户友好、可操作）：
> "testing" skill 已被发现但未加载，因为文件夹名称和 skill 文件之间存在名称不匹配：
>
> | | 值 |
> |---|---|
> | **文件夹名称** | `testing` |
> | **SKILL.md 中的名称** | `testing2` |
>
> 两者必须匹配才能加载 skill。
>
> **修复方法**
>
> 选择以下任一方式：
> - 将 SKILL.md 中的 `name: testing2` 改为 `name: testing`，**或**
> - 将文件夹从 `testing/` 重命名为 `testing2/`
>
> 然后开启新的 chat session 以使其生效。

## 重要规则

- 永远不要在没有证据的情况下假设因果关系。
- 使用 `run_in_terminal` 搜索日志文件 — 永远不要使用 `grep_search`（它无法访问 workspace 之外的文件）。macOS/Linux 使用 `grep`/`jq`，Windows 使用 `Select-String`/`node -e`。
- 永远不要用 `read_file` 读取整个日志文件 — 它们可能非常大。先搜索，然后对小的目标范围使用 `read_file`。
- 保持日志访问的针对性和高效性。
- 如果怀疑网络问题，在得出结论前，使用 `run_vscode_command` 工具运行 `github.copilot.debug.collectDiagnostics` 并使用返回的诊断字符串。
- 如果未找到明确原因，在放弃前查阅 Copilot Issues wiki。如果 wiki 也没有相关信息，明确告知用户。
