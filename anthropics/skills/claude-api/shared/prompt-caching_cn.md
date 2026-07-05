# Prompt 缓存 — 设计与优化

本文件介绍如何设计 prompt 构建代码以实现高效缓存。如需各语言的语法示例，请参见各语言 README 或单文件文档中的 `## Prompt Caching` 章节。

## 一切规则的核心不变量

**Prompt 缓存是前缀匹配。前缀中任何位置的任何变更都会使其后的所有内容失效。**

缓存键由每个 `cache_control` 断点之前渲染后 prompt 的精确字节推导而来。位置 N 处的单字节差异——时间戳、重新排序的 JSON 键、列表中的不同工具——都会使位置 ≥ N 的所有断点的缓存失效。

渲染顺序为：`tools` → `system` → `messages`。在最后一个 system 块上设置断点会同时缓存 tools 和 system。

围绕这一约束来设计 prompt 构建路径。排序正确，大部分缓存自然生效。排序错误，再多的 `cache_control` 标记也无济于事。

---

## 优化现有代码的工作流程

当被要求添加或优化缓存时：

1. **追踪 prompt 组装路径。** 找到 `system`、`tools` 和 `messages` 的构建位置。识别流入其中的每个输入。
2. **按稳定性对每个输入进行分类：**
   - 永不变化 → 放在 prompt 前部，在任何断点之前
   - 按会话变化 → 放在全局前缀之后，按会话缓存
   - 按轮次变化 → 放在末尾，在最后一个断点之后
   - 按请求变化（时间戳、UUID、随机 ID）→ **消除或移至最末尾**
3. **检查渲染顺序是否与稳定性顺序一致。** 稳定内容必须在物理上位于易变内容之前。如果时间戳被插入到 system prompt 头部，则其后的所有内容都无法缓存，无论标记如何。
4. **在稳定性边界处放置断点。** 参见下方的放置模式。
5. **审查静默失效因素。** 参见反模式表格。

---

## 放置模式

### 跨多个请求共享的大型 system prompt

在最后一个 system 文本块上放置断点。如果有 tools，它们在 system 之前渲染——最后一个 system 块上的标记会同时缓存 tools + system。

```json
"system": [
  {"type": "text", "text": "<large shared prompt>", "cache_control": {"type": "ephemeral"}}
]
```

### 多轮对话

在最近追加轮次的最后一个内容块上放置断点。每个后续请求都会复用整个先前的对话前缀。较早的断点仍然是有效的读取点，因此命中数会随着对话增长而逐步累积。

```json
// 最后一个 user 轮次的最后一个内容块
messages[-1].content[-1].cache_control = {"type": "ephemeral"}
```

### 共享前缀，变化后缀

许多请求共享一个大型固定前导部分（few-shot 示例、检索文档、指令），但在最终问题上有所不同。将断点放在**共享**部分的末尾，而非整个 prompt 的末尾——否则每个请求都会写入不同的缓存条目，永远不会被读取命中。

```json
"messages": [{"role": "user", "content": [
  {"type": "text", "text": "<shared context>", "cache_control": {"type": "ephemeral"}},
  {"type": "text", "text": "<varying question>"}  // 无标记——每次不同
]}]
```

### 会话中途 system 消息

**仅限 Claude Opus 4.8；无需 beta header。** 当会话中途收到操作指令时——模式切换、更新上下文、动态注入状态——将其作为 `{"role": "system", "content": "..."}` 追加到 `messages[]`，而非编辑顶层 `system`。编辑顶层 `system` 会改变整个对话历史之前的前缀，导致每个已缓存的轮次都被重新处理（无缓存命中）；而 `role: "system"` 消息位于历史之后，保持已缓存的前缀不变。

```json
// 顶层 system 保持字节相同；新指令放在已缓存历史之后
"system": [{"type": "text", "text": "<stable core>", "cache_control": {"type": "ephemeral"}}],
"messages": [
  ...history,
  {"role": "user", "content": "..."},
  {"role": "system", "content": "Terse mode enabled — keep responses under 40 words."}
]
```

这也是在 user 轮次中以文本形式嵌入操作指令（`<system-reminder>` 模式）的防注入替代方案：两者具有相同的缓存特性，但 `role: "system"` 是不可伪造的操作通道，而 user/tool 内容中的文本可以被任何写入用户可见输入的内容伪造。

适用于 Claude Opus 4.8；无需 beta header。必须跟在 `role: "user"` 消息之后（或以服务端 tool use 结束的 `assistant` 消息之后），且必须是 `messages` 中的最后一条或其后跟一个 `assistant` 轮次；不能是 `messages[0]`——初始 prompt 请使用顶层 `system`。内容仅限文本。不支持的模型会返回 400（`BadRequestError`：`role 'system' is not supported on this model`）；捕获该错误并回退到将指令放入 user 轮次的 `<system-reminder>` 块中。

### 每次从头变化的 prompt

不要缓存。如果每次请求的前 1K token 都不同，则没有可复用的前缀。添加 `cache_control` 只会白白支付缓存写入溢价而没有任何读取。不要添加。

---

## 架构指导

以下决策比标记放置更重要。先修正这些。

**保持 system prompt 不变。** 不要在 system prompt 中插入"当前日期：X"、"模式：Y"、"用户名：Z"——它们位于前缀最前面，会使后续所有内容失效。将动态上下文注入到 `messages` 中更靠后的位置——作为 `{"role": "system", ...}` 消息（在支持的模型上，参见上方"会话中途 system 消息"章节），或作为 user 消息中的文本。第 5 轮的一条消息不会使第 5 轮之前的任何内容失效。

**不要在对话中途更改 tools 或模型。** Tools 在位置 0 渲染；添加、删除或重新排序 tool 会使整个缓存失效。切换模型也是如此（缓存是按模型隔离的）。如果需要"模式"，不要交换 tool 集——给 Claude 一个记录模式切换的 tool，或将模式作为消息内容传递。确定性地序列化工具（按名称排序）。

**分叉操作必须复用父级的精确前缀。** 辅助计算（摘要、压缩、子 agent）通常会启动单独的 API 调用。如果分叉重建 `system` / `tools` / `model` 时有任何差异，就会完全错过父级的缓存。逐字复制父级的 `system`、`tools` 和 `model`，然后在末尾追加分叉特定的内容。

---

## 静默失效因素

审查代码时，在送入 prompt 前缀的所有内容中搜索以下模式：

| 模式 | 为何会破坏缓存 |
|---|---|
| `datetime.now()` / `Date.now()` / `time.time()` 出现在 system prompt 中 | 每次请求前缀都不同 |
| `uuid4()` / `crypto.randomUUID()` / 请求 ID 出现在内容前部 | 同上——每次请求都唯一 |
| `json.dumps(d)` 未使用 `sort_keys=True` / 迭代 `set` | 非确定性序列化 → 前缀字节不同 |
| f-string 将 session/user ID 插入 system prompt | 按用户的前缀；无法跨用户共享 |
| 条件 system 区段（`if flag: system += ...`） | 每种 flag 组合都是不同的前缀 |
| `tools=build_tools(user)` 其中工具集按用户变化 | Tools 在位置 0 渲染；无法跨用户缓存 |

修复方法：将动态部分移到最后一个断点之后，使其确定性化，或在不承担关键作用时直接删除。

---

## API 参考

```json
"cache_control": {"type": "ephemeral"}              // 5 分钟 TTL（默认）
"cache_control": {"type": "ephemeral", "ttl": "1h"} // 1 小时 TTL
```

- 每个请求最多 **4** 个 `cache_control` 断点。
- 可放在任何内容块上：system 文本块、tool 定义、消息内容块（`text`、`image`、`tool_use`、`tool_result`、`document`）。
- `messages.create()` 上的顶层 `cache_control` 会自动放置在最后一个可缓存块上——当你不需要精细放置时，这是最简单的选项。
- 最小可缓存前缀取决于模型。较短的前缀即使有标记也不会缓存——不会报错，只会显示 `cache_creation_input_tokens: 0`：

| 模型 | 最小值 |
|---|---:|
| Opus 4.8、Opus 4.7、Opus 4.6、Opus 4.5、Haiku 4.5 | 4096 token |
| Fable 5、Sonnet 4.6、Haiku 3.5、Haiku 3 | 2048 token |
| Sonnet 4.5、Sonnet 4.1、Sonnet 4、Sonnet 3.7 | 1024 token |

一个 3K token 的 prompt 在 Sonnet 4.5 和 Fable 5 上可以缓存，但在 Opus 4.8 上会静默失败。

**经济性：** 缓存读取成本约为基础输入价格的 0.1 倍。缓存写入成本为 **5 分钟 TTL 的 1.25 倍，1 小时 TTL 的 2 倍**。收支平衡取决于 TTL：5 分钟 TTL 下，两次请求即可收支平衡（1.25 倍 + 0.1 倍 = 1.35 倍 vs 2 倍无缓存）；1 小时 TTL 下，至少需要三次请求（2 倍 + 0.2 倍 = 2.2 倍 vs 3 倍无缓存）。1 小时 TTL 可在突发流量的间隙保持条目存活，但翻倍的写入成本意味着需要更多读取才能回本。

---

## 验证缓存命中

响应的 `usage` 对象报告缓存活动：

| 字段 | 含义 |
|---|---|
| `cache_creation_input_tokens` | 本次请求写入缓存的 token（你支付了约 1.25 倍的写入溢价） |
| `cache_read_input_tokens` | 本次请求从缓存读取的 token（你支付了约 0.1 倍） |
| `input_tokens` | 以全价处理的 token（未缓存） |

如果在具有相同前缀的重复请求中 `cache_read_input_tokens` 始终为零，说明存在静默失效因素——对比两次请求的渲染 prompt 字节来找出原因。

**`input_tokens` 仅是未缓存的剩余部分。** 总 prompt 大小 = `input_tokens + cache_creation_input_tokens + cache_read_input_tokens`。如果你的 agent 运行了数小时但 `input_tokens` 只显示 4K，其余都是从缓存服务的——检查总和，而非单个字段。

各语言的访问方式：`response.usage.cache_read_input_tokens`（Python/TS/Ruby）、`$message->usage->cacheReadInputTokens`（PHP）、`resp.Usage.CacheReadInputTokens`（Go/C#）、`.usage().cacheReadInputTokens()`（Java）。

---

## 失效层级

并非每个参数变更都会使所有内容失效。API 有三个缓存层级，变更只会使其自身层级及以下的缓存失效：

| 变更 | Tools 缓存 | System 缓存 | Messages 缓存 |
|---|:---:|:---:|:---:|
| Tool 定义（添加/删除/重新排序） | ❌ | ❌ | ❌ |
| 模型切换 | ❌ | ❌ | ❌ |
| `speed`、web-search、citations 开关 | ✅ | ❌ | ❌ |
| System prompt 内容 | ✅ | ❌ | ❌ |
| `tool_choice`、images、`thinking` 启用/禁用 | ✅ | ✅ | ❌ |
| Message 内容 | ✅ | ✅ | ❌ |

含义：你可以在每次请求时更改 `tool_choice` 或切换 `thinking` 而不丢失 tools+system 缓存。不必过度担心这些——只有 tool 定义和模型变更才会强制完全重建。

---

## 20 块回溯窗口

每个断点向后回溯**最多 20 个内容块**来查找先前的缓存条目。如果单个轮次添加超过 20 个块（在具有大量 tool_use/tool_result 对的 agentic 循环中很常见），下一个请求的断点将找不到先前的缓存并静默未命中。

修复方法：在长轮次中每约 15 个块放置一个中间断点，或将标记放在距上一轮最后一个已缓存块 20 个以内的块上。

---

## 并发请求时序

缓存条目仅在第一个响应**开始流式传输**后才可读。N 个具有相同前缀的并行请求都需支付全价——没有一个能读取其他请求正在写入的内容。

对于扇出模式：发送 1 个请求，等待第一个流式 token（不是完整响应），然后发出剩余的 N−1 个。它们将读取第一个请求刚写入的缓存。

## 预热缓存

为了消除*首次*真实请求的缓存未命中延迟，在启动时（或按间隔）发送一个 **`max_tokens: 0`** 请求。API 执行预填充——在你指定的 `cache_control` 断点处写入缓存——然后立即返回 `content: []`、`stop_reason: "max_tokens"` 和填充好的 `usage` 块（零输出 token 计费；`cache_creation_input_tokens` 按正常缓存写入收费）。

**何时预热** — 预热是用*现在*的缓存写入费用换取*下一次*真实请求的更低 TTFT。当以下三个条件同时满足时值得这样做：(a) 首次请求延迟是用户可感知的（聊天/语音/交互式——非后台任务），(b) 共享前缀足够大以至于冷写入明显缓慢，(c) 在流量到来*之前*有时间发送——应用启动、worker 启动、部署后、定时窗口开始。

| 跳过预热的情况… | 原因 |
|---|---|
| 流量是连续的（请求间隔 ≤ TTL） | 第一次真实请求就会预热缓存，后续每次都会命中；单独的预热调用是纯粹的额外写入 |
| 前缀很小或低于可缓存最小值 | 冷写入的惩罚可以忽略不计 |
| 前缀按请求/用户变化 | 没有可预热的共享内容 |
| 你会投机性地预热许多不同的前缀 | 每个都是约 1.25 倍的写入；成本可能超过你节省的延迟 |

**定时重新预热：** 仅在流量有超过 TTL 的间隙时才需要。如果真实请求的到达频率高于每 5 分钟一次，它们会自行保持缓存温暖——不要添加定间隔重新预热。对于有长空闲间隙的突发流量，要么在略低于 TTL 的时间重新预热，要么切换到 `ttl: "1h"` 并减少重新预热频率。

```python
client.messages.create(
    model="claude-opus-4-8",
    max_tokens=0,
    system=[{
        "type": "text",
        "text": SYSTEM_PROMPT,
        "cache_control": {"type": "ephemeral"},
    }],
    messages=[{"role": "user", "content": "warmup"}],
)
```

**断点放置：** 将 `cache_control` 放在**与真实请求共享的最后一个块**（system prompt 或 tool 定义）上——**不要**放在占位 user 消息上，也**不要**通过顶层自动缓存（这会将缓存键绑定到占位符上）。占位符可以是任何非空白字符串；它在预填充期间被读取但永远不会被回答。

**不允许的组合：** `max_tokens: 0` 与 `stream: true`、`thinking.type: "enabled"`、`output_config.format`、`tool_choice` 为 `{"type":"tool"}` 或 `{"type":"any"}`、或在 Message Batches 请求中组合使用时，会返回 `invalid_request_error`。

**TTL 仍然有效** — 默认缓存至少每 5 分钟重新预热一次，或使用 1 小时 TTL。这替代了旧的 `max_tokens: 1` 变通方案（无需丢弃单 token 回复，无输出 token 计费，意图明确）。