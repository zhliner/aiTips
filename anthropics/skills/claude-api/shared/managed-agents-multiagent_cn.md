# Managed Agents — 多 Agent 会话

协调者 Agent 可以在一个会话中将任务委派给其他 Agent。所有 Agent **共享容器和文件系统**；每个 Agent 在自己的 **线程** 中运行——一个上下文隔离的事件流，拥有独立的对话历史、模型、系统提示、工具、MCP 服务器和技能（来自该 Agent 自身的配置）。线程是持久化的：协调者可以向之前调用过的子 Agent 发送后续消息，该子 Agent 会保留其先前的对话轮次。

SDK 会在所有 `client.beta.{agents,sessions}.*` 调用中自动设置 `managed-agents-2026-04-01` beta header；多 Agent 功能无需额外的 header。

---

## 在协调者上声明名册

`multiagent` 是 `agents.create()` / `agents.update()` 上的**顶级字段**——**不是** `tools[]` 条目。`agents` 列出 1–20 个名册条目。`sessions.create()` 无需任何更改——名册从协调者的配置中解析。

```python
orchestrator = client.beta.agents.create(
    name="Engineering Lead",
    model="claude-opus-4-8",
    system="You coordinate engineering work. Delegate code review to the reviewer and test writing to the test agent.",
    tools=[{"type": "agent_toolset_20260401"}],
    multiagent={
        "type": "coordinator",
        "agents": [
            reviewer.id,                                            # 纯字符串——最新版本
            {"type": "agent", "id": test_writer.id, "version": 4},  # 固定版本
            {"type": "self"},                                       # 协调者自身
        ],
    },
)

session = client.beta.sessions.create(agent=orchestrator.id, environment_id=env.id)
```

| 名册条目 | 形式 | 说明 |
|---|---|---|
| 字符串简写 | `"agent_abc123"` | 引用已存储 Agent 的最新版本。 |
| Agent 引用 | `{type: "agent", id, version?}` | 省略 `version` 以在协调者保存时固定最新版本。 |
| 自身 | `{type: "self"}` | 协调者可以生成自身的副本。 |

如果会话是通过 `agent_with_overrides` 创建的（参见 `shared/managed-agents-core.md` → 为会话覆盖 Agent 配置），这些覆盖适用于**协调者及其 `self` 副本**。通过 ID 引用的名册 Agent 始终使用其创建时的配置——覆盖不会传播到它们。

名册中最多 **20 个唯一 Agent**；协调者可以生成每个 Agent 的**多个副本**。**仅支持一级委派**——深度 > 1 会被忽略。

---

## 线程

会话级事件流是**主线程**——它显示协调者的追踪以及子 Agent 活动的概览（线程状态转换和跨线程消息，而非每个子 Agent 的工具调用）。通过每线程端点深入查看特定子 Agent：

| 操作 | HTTP | SDK (`client.beta.sessions.threads.*`) |
|---|---|---|
| 列出线程 | `GET /v1/sessions/{sid}/threads` | `.list(session_id)` |
| 检索单个 | `GET /v1/sessions/{sid}/threads/{tid}` | `.retrieve(thread_id, session_id=...)` |
| 归档 | `POST /v1/sessions/{sid}/threads/{tid}/archive` | `.archive(thread_id, session_id=...)` |
| 列出线程事件 | `GET /v1/sessions/{sid}/threads/{tid}/events` | `.events.list(thread_id, session_id=...)` |
| 流式传输线程事件 | `GET /v1/sessions/{sid}/threads/{tid}/stream` | `.events.stream(thread_id, session_id=...)` |

每个 `SessionThread` 包含 `id`、`status`（`running` | `idle` | `rescheduling` | `terminated`）、`agent`（Agent 配置的解析快照——`id`、`name`、`model`、`system`、`tools`、`skills`、`mcp_servers`、`version`）、`parent_thread_id`（主线程为 null，包含在列表中）、`archived_at`，以及可选的 `stats`/`usage`。**会话状态聚合线程状态**——如果任何线程为 `running`，则 `session.status` 为 `running`。最多 **25 个并发线程**。排空每线程流时，在收到 `session.thread_status_idle` 时中断（并像处理会话级 idle 一样检查其 `stop_reason`）。

---

## 多 Agent 事件（会话流上）

| 事件 | 有效载荷要点 | 含义 |
|---|---|---|
| `session.thread_created` | `session_thread_id`、`agent_name` | 新线程已创建。 |
| `session.thread_status_running` | `session_thread_id`、`agent_name` | 线程开始活动。 |
| `session.thread_status_idle` | `session_thread_id`、`agent_name`、**`stop_reason`** | 线程等待输入。检查 `stop_reason`（与 `session.status_idle.stop_reason` 结构相同）。 |
| `session.thread_status_rescheduled` | `session_thread_id`、`agent_name` | 线程在可重试错误后重新调度。 |
| `session.thread_status_terminated` | `session_thread_id`、`agent_name` | 线程已归档或遇到终止错误。 |
| `agent.thread_message_sent` | `to_session_thread_id`、`to_agent_name`、`content` | 协调者向另一个线程发送了后续消息。 |
| `agent.thread_message_received` | `from_session_thread_id`、`from_agent_name`、`content` | Agent 将其结果传递给协调者。 |

---

## 子 Agent 线程的工具权限和自定义工具

当子 Agent 需要你的客户端响应时（`always_ask` 确认或自定义工具结果），请求会**跨线程发送到主线程**，并带有 `session_thread_id` 标识源线程——因此你只需监听会话流。使用 `user.tool_confirmation`（携带 `tool_use_id`）或 `user.custom_tool_result`（携带 `custom_tool_use_id`）回复，并**回显源事件中的 `session_thread_id`**（SDK 参数类型和文档字符串期望如此）。服务器也通过工具使用 ID 进行路由，因此回显是双重保险而非关键依赖——但仍需包含它。

```python
for event_id in stop.event_ids:
    pending = events_by_id[event_id]
    confirmation = {
        "type": "user.tool_confirmation",
        "tool_use_id": event_id,
        "result": "allow",
    }
    if pending.session_thread_id is not None:
        confirmation["session_thread_id"] = pending.session_thread_id
    client.beta.sessions.events.send(session.id, events=[confirmation])
```

同样的模式适用于 `user.custom_tool_result`。

---

## 常见陷阱

- **不要将名册放在 `sessions.create()` 或 `tools[]` 中。** `multiagent` 是 Agent 的顶级字段；先更新协调者，再启动引用它的会话。
- **不要假设共享上下文。** 线程共享文件系统，但不共享对话历史或工具。如果协调者需要子 Agent 对某事物执行操作，必须在委派消息中说明（或将其写入磁盘）。
- **深度 > 1 会被忽略。** 子 Agent 自身的 `multiagent` 名册（如果有）不会级联——只有会话的协调者进行委派。

Python 以外语言的绑定详情，请参阅 `shared/live-sources.md` 中的 `https://platform.claude.com/docs/en/managed-agents/multi-agent.md`。
