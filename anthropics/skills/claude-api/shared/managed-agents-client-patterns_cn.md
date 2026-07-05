# Managed Agents — 常用客户端模式

在驱动 Managed Agent 会话时，客户端侧需要编写的常见模式，基于可用的 SDK 示例。

代码示例使用 TypeScript — 其他语言遵循相同的模式；参见 `{lang}/managed-agents/README.md`（cURL 和 C#: `curl/managed-agents.md`）获取对应示例。

---

## 1. 无损流重连

**问题：** SSE 没有重放机制。如果连接在会话中途断开，简单的重连会从"当前时刻"重新打开流，你将默默丢失中间发出的所有事件。

**解决方案：** 重连时，在消费实时流*之前*，先通过 `events.list()` 获取完整的事件历史，并在实时流追平时按事件 ID 去重。

```ts
const seenEventIds = new Set<string>()
const stream = await client.beta.sessions.events.stream(session.id)

// 流已打开，服务端正在缓冲。先读取历史。
for await (const event of client.beta.sessions.events.list(session.id)) {
  seenEventIds.add(event.id)
  handle(event)
}

// 跟踪实时流。去重仅控制 handle() — 终止检查必须对已见过的事件也执行，
// 否则历史响应中的终止事件会被 `continue` 跳过，循环永远不会退出。
for await (const event of stream) {
  if (!seenEventIds.has(event.id)) {
    seenEventIds.add(event.id)
    handle(event)
  }
  if (event.type === 'session.status_terminated') break
  if (event.type === 'session.status_idle' && event.stop_reason.type !== 'requires_action') break
}
```

---

## 2. `processed_at` — 已排队 vs 已处理

流上的每个事件都携带 `processed_at`（ISO 8601）。对于客户端发送的事件（`user.message`、`user.interrupt`、`user.tool_confirmation`、`user.custom_tool_result`），当事件已排队但尚未被 Agent 处理时为 `null`，Agent 处理后则填充时间戳。同一事件会在流上出现两次 — 一次 `processed_at: null`，一次带有时间戳。

```ts
for await (const event of stream) {
  if (event.type === 'user.message') {
    if (event.processed_at == null) onQueued(event.id)
    else onProcessed(event.id, event.processed_at)
  }
}
```

使用此机制驱动你发送的所有内容的 pending → acknowledged UI 状态。如何将本地渲染的乐观消息映射到服务端分配的 `event.id` 取决于具体应用（通常通过 `events.send()` 的返回值或 FIFO 顺序）。

---

## 3. 中断正在运行的会话

发送 `user.interrupt` 作为普通事件。会话继续运行直到到达安全边界，然后进入 idle 状态。

```ts
await client.beta.sessions.events.send(session.id, {
  events: [{ type: 'user.interrupt' }],
})

// 持续消费直到会话真正结束 — 完整判断条件参见模式 5。
for await (const event of stream) {
  if (event.type === 'session.status_terminated') break
  if (
    event.type === 'session.status_idle' &&
    event.stop_reason.type !== 'requires_action'
  ) break
}
```

参考：`interrupt.ts` — 在检测到 `span.model_request_start` 时立即发送中断，消费到 idle 状态，然后通过 `sessions.retrieve()` 验证。

---

## 4. `tool_confirmation` 往返

当 Agent 配置了 `permission_policy: { type: 'always_ask' }` 时，对该工具的任何调用都会触发 `agent.tool_use` 事件，且 `evaluated_permission === 'ask'`，会话进入 idle 等待决策。使用 `user.tool_confirmation` 响应。

```ts
for await (const event of stream) {
  if (event.type === 'agent.tool_use' && event.evaluated_permission === 'ask') {
    await client.beta.sessions.events.send(session.id, {
      events: [{
        type: 'user.tool_confirmation',
        tool_use_id: event.id,         // 不是 toolu_ id — 使用 event.id
        result: 'allow',               // 或 'deny'
        // deny_message: '...',        // 可选，仅在 result: 'deny' 时使用
      }],
    })
  }
}
```

关键要点：
- `tool_use_id` 是 `event.id`（通常为 `sevt_...`），**不是** `toolu_...` ID。
- `result` 为 `'allow' | 'deny'`。使用 `deny_message` 告诉模型*为什么*拒绝 — 它会回传给 Agent。
- 多个待处理工具：对每个 `evaluated_permission === 'ask'` 的 `agent.tool_use` 事件分别响应一次。

参考：`tool-permissions.ts`。

---

## 5. 正确的 idle 退出判断

不要仅凭 `session.status_idle` 就退出。会话会暂时进入 idle — 例如在并行工具执行之间、等待 `user.tool_confirmation` 时、或等待 `user.custom_tool_result` 时。仅在 idle 且带有终止性 `stop_reason` 时退出，或在 `session.status_terminated` 时退出。

```ts
for await (const event of stream) {
  handle(event)
  if (event.type === 'session.status_terminated') break
  if (event.type === 'session.status_idle') {
    if (event.stop_reason.type === 'requires_action') continue // 等待你的操作 — 处理它
    break // end_turn 或 retries_exhausted — 两者都是终止性的
  }
}
```

`session.status_idle` 上的 `stop_reason.type` 值：
- `requires_action` — Agent 正在等待客户端事件（工具确认、自定义工具结果）。处理它，不要退出。
- `retries_exhausted` — 终止性失败。退出，然后检查 `sessions.retrieve()` 获取错误状态。
- `end_turn` — 正常完成。

---

## 6. idle 后的状态写入竞态

SSE 流发出 `session.status_idle` 的时间略早于会话的可查询状态反映该变化。在 idle 时立即退出并调用 `sessions.delete()` 或 `sessions.archive()` 的客户端会间歇性地收到 400 错误 "cannot delete/archive while running"。

在清理前轮询：

```ts
let s
for (let i = 0; i < 10; i++) {
  s = await client.beta.sessions.retrieve(session.id)
  if (s.status !== 'running') break
  await new Promise(r => setTimeout(r, 200))
}
if (s?.status !== 'running') {
  await client.beta.sessions.archive(session.id)
} // 否则：2 秒后仍在运行 — 不要归档，让它稳定或上报问题
```

---

## 7. 先开流，再发送

始终**先打开流**，再发送启动事件。否则 Agent 可能在你的消费者连接之前就处理了事件并发出了前几个事件，你会错过它们。

```ts
const stream = await client.beta.sessions.events.stream(session.id)
await client.beta.sessions.events.send(session.id, {
  events: [{ type: 'user.message', content: [{ type: 'text', text: 'Hello' }] }],
})
for await (const event of stream) { /* ... */ }
```

`Promise.all([stream, send])` 的写法也可以，但先开流更简单且效果相同 — 流在打开的瞬间就开始缓冲。

---

## 8. 文件挂载注意事项

**挂载的资源与上传的文件有不同的 `file_id`。** 会话创建时会创建一个会话范围内的副本。

```ts
const uploaded = await client.beta.files.upload({ file })
// uploaded.id         → 原始文件
const session = await client.beta.sessions.create({
  /* ... */
  resources: [{ type: 'file', file_id: uploaded.id, mount_path: '/workspace/data.csv' }],
})
// session.resources[0].file_id !== uploaded.id  ← 不同的 ID
```

通过 `files.delete(uploaded.id)` 删除原始文件；会话范围内的副本随会话一起被垃圾回收。`mount_path` 必须为绝对路径 — 参见 `shared/managed-agents-environments.md`。

---

## 9. 非 MCP API 和 CLI 的密钥 — 通过自定义工具保持在主机侧

**问题：** 你希望 Agent 调用需要密钥（API 密钥、token、服务账号凭证）的第三方 API 或运行 CLI，但你不能或不想将密钥交给 vault。

**优先检查：** 对于云环境，首选方案现在是 vault 的 `environment_variable` 凭证 — Agent 的 shell 看到的是不透明的占位符，真实密钥在出口处替换。参见 `shared/managed-agents-tools.md` → Vaults。在以下情况下改用此模式：**自托管沙箱**（环境变量凭证尚不支持）、拒绝占位符的客户端（通过本地格式验证）、绝不能离开你的基础设施的密钥、或需要主机侧二进制文件的调用。

**解决方案：** 将认证调用移到你的侧。在 Agent 上声明自定义工具；当 Agent 发出 `agent.custom_tool_use` 时，你的编排器（读取 SSE 流的进程）使用自己的凭证执行调用，并通过 `user.custom_tool_result` 响应。容器永远看不到密钥。

```ts
// Agent 模板：声明工具，不包含凭证
tools: [{ type: 'custom', name: 'linear_graphql', input_schema: { /* query, vars */ } }]

// 编排器：使用主机侧凭证处理调用
for await (const event of stream) {
  if (event.type === 'agent.custom_tool_use' && event.name === 'linear_graphql') {
    const result = await linear.request(event.input.query, event.input.vars) // 主机的密钥
    await client.beta.sessions.events.send(session.id, {
      events: [{ type: 'user.custom_tool_result', tool_use_id: event.id, result }],
    })
  }
}
```

同样的模式适用于 `gh` CLI、本地评估脚本，或任何需要主机侧认证或二进制文件的操作。

**安全说明：** 这不会暴露公共端点。`agent.custom_tool_use` 通过你的编排器已使用 Anthropic API 密钥保持打开的 SSE 流到达，`user.custom_tool_result` 通过 `events.send()` 在同一密钥下返回。你的编排器是客户端，不是服务器 — 没有未认证的监听。

**不要将 API 密钥嵌入系统提示词或用户消息中作为变通方案。** 提示词和消息存储在会话的事件历史中，通过 `events.list()` 返回，并包含在压缩摘要中 — 放在那里的密钥会被持久保存，并在会话生命周期内通过 API 可读。