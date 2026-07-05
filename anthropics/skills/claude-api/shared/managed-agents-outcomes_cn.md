# Managed Agents — Outcome

**Outcome** 将会话从*对话*提升为*工作*：你声明"完成"的标准，框架运行 迭代 → 评分 → 修订 循环，直到产出物满足评分标准、达到 `max_iterations`，或被中断。一个独立的**评分器**（独立上下文窗口）根据评分标准对每次迭代打分，并将每个标准的差距反馈给 Agent。

SDK 会在所有 `client.beta.sessions.*` 调用中自动设置 `managed-agents-2026-04-01` beta header；Outcome 功能无需额外的 header。

---

## `user.define_outcome` 事件

Outcome 不是 `sessions.create()` 上的字段。你创建一个正常会话，然后发送 `user.define_outcome` 事件。Agent 在收到事件后开始工作——**不要同时发送 `user.message`** 来启动它。

```python
session = client.beta.sessions.create(
    agent=AGENT_ID,
    environment_id=ENVIRONMENT_ID,
    title="Financial analysis on Costco",
)

client.beta.sessions.events.send(
    session_id=session.id,
    events=[
        {
            "type": "user.define_outcome",
            "description": "Build a DCF model for Costco in .xlsx",
            "rubric": {"type": "text", "content": RUBRIC_MD},
            # 或: "rubric": {"type": "file", "file_id": rubric.id}
            "max_iterations": 5,  # 可选；默认 3，最大 20
        }
    ],
)
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `type` | `"user.define_outcome"` | |
| `description` | string | 任务。这是 Agent 努力达成的目标——无需单独的 `user.message`。 |
| `rubric` | `{type: "text", content}` \| `{type: "file", file_id}` | **必填。** 包含明确的、可独立评分的标准的 Markdown。通过 `client.beta.files.upload(...)`（beta `files-api-2025-04-14`）上传一次，以便跨会话复用。 |
| `max_iterations` | int | 可选。默认 **3**，最大 **20**。 |

该事件会在流上回显，附带服务器分配的 `outcome_id` 和 `processed_at`。

> **编写评分标准。** 使用明确的、可评分的标准（"CSV 有一个数值类型的 `price` 列"），而非模糊的描述（"数据看起来不错"）——评分器独立评分每个标准，因此模糊的标准会产生嘈杂的循环。如果你没有评分标准，可以让 Claude 分析一个已知合格的产出物，并将该分析转化为评分标准。

---

## Outcome 专用事件

这些事件出现在标准事件流（`sessions.events.stream` / `.list`）上，与通常的 `agent.*` / `session.*` 事件并列。

| 事件 | 有效载荷要点 | 含义 |
|---|---|---|
| `span.outcome_evaluation_start` | `outcome_id`、`iteration`（从 0 开始） | 评分器开始对第 *N* 次迭代评分。 |
| `span.outcome_evaluation_ongoing` | `outcome_id` | 评分器运行中的心跳。评分器的推理过程不透明——你只能看到*它在*工作，而非*它在想*什么。 |
| `span.outcome_evaluation_end` | `outcome_evaluation_start_id`、`outcome_id`、`iteration`、`result`、`explanation`、`usage` | 评分器完成一次迭代。`result` 决定接下来的行为（见下表）。 |

### `span.outcome_evaluation_end.result`

| `result` | 后续行为 |
|---|---|
| `satisfied` | 会话 → `idle`。该 Outcome 终止。 |
| `needs_revision` | Agent 开始新一轮迭代。 |
| `max_iterations_reached` | 不再有评分器循环。Agent 可能运行最后一次修订，然后会话 → `idle`。 |
| `failed` | 会话 → `idle`。评分标准从根本上与任务不匹配（例如 description 和 rubric 矛盾）。 |
| `interrupted` | 仅在 `_start` 已触发后收到 `user.interrupt` 时才发出。 |

```json
{
  "type": "span.outcome_evaluation_end",
  "id": "sevt_01jkl...",
  "outcome_evaluation_start_id": "sevt_01def...",
  "outcome_id": "outc_01a...",
  "result": "satisfied",
  "explanation": "All 12 criteria met: revenue projections use 5 years of historical data, ...",
  "iteration": 0,
  "usage": { "input_tokens": 2400, "output_tokens": 350, "cache_creation_input_tokens": 0, "cache_read_input_tokens": 1800 },
  "processed_at": "2026-03-25T14:03:00Z"
}
```

---

## 检查状态与获取产出物

**状态**——可以监听流中的 `span.outcome_evaluation_end`，或轮询会话并读取 `outcome_evaluations`：

```python
session = client.beta.sessions.retrieve(session.id)
for ev in session.outcome_evaluations:
    print(f"{ev.outcome_id}: {ev.result}")  # outc_01a...: satisfied
```

**产出物**——Agent 将结果写入 `/mnt/session/outputs/`。会话空闲后，通过 Files API 使用 `scope_id=session.id` 获取。这与 `shared/managed-agents-environments.md` → 会话产出物 中记录的会话产出物机制相同（包括 `files.list` 上的双 beta header 要求）。

---

## 交互规则与常见陷阱

- **一次一个 Outcome。** 在上一个 Outcome 的终止 `span.outcome_evaluation_end`（`satisfied` / `max_iterations_reached` / `failed` / `interrupted`）之后，才发送下一个 `user.define_outcome` 进行链式调用。会话在链式 Outcome 之间保留历史。
- **允许中途引导但非必需。** 你*可以*在 Outcome 进行中途发送 `user.message` 事件来引导方向，但 Agent 已经知道持续工作直到终止——不要发送"继续"提示。
- **`user.interrupt` 暂停当前 Outcome**——它标记 `result: "interrupted"` 并使会话变为 `idle`，准备好接受新的 Outcome 或对话轮次。
- **终止后，会话可复用**——继续对话或定义新的 Outcome。
- **Outcome ≠ 会话创建字段。** 不要在 `sessions.create()` 上放置 `outcome`、`rubric` 或 `description`——Outcome 始终作为 `user.define_outcome` 事件发送。
- **Idle 中断条件不变。** 在排空循环中，继续使用 `event.type === 'session.status_idle' && event.stop_reason?.type !== 'requires_action'`——**不要**仅以 `span.outcome_evaluation_end` 作为条件（在 `needs_revision` 时会话继续运行）。参见 `shared/managed-agents-client-patterns.md` 模式 5。

原始 HTTP 结构和 Python 以外语言的 SDK 绑定详情，请参阅 `shared/live-sources.md` 中的 `https://platform.claude.com/docs/en/managed-agents/define-outcomes.md`。
