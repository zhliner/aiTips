# Managed Agents — 定时部署

**定时部署**按周期性的 cron 计划运行 Agent —— 每次触发都会自主创建一个会话。适用于固定节奏的工作：夜间分类、每周合规扫描、每小时监控。

需要 `managed-agents-2026-04-01` beta header（SDK 会在 `client.beta.deployments.*` / `client.beta.deployment_runs.*` 调用时自动设置）。

## 创建部署

部署打包了会话所需的一切（Agent、环境、可选的文件 / GitHub / memory store / Vault），加上 `schedule` 和每次运行启动时的 `initial_events`：

- `agent` 和 `environment_id` 为必填 —— 与 `sessions.create` 的结构相同（参见 `shared/managed-agents-core.md`）。
- `initial_events` 必须包含起始的 `user.message`。
- `schedule` 接受 cron `expression` 和 IANA `timezone`。最大精度为分钟级。

```bash
curl -fsSL https://api.anthropic.com/v1/deployments \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: managed-agents-2026-04-01" \
  -H "content-type: application/json" \
  -d @- <<EOF
{
  "name": "Weekly compliance scan",
  "agent": "$AGENT_ID",
  "environment_id": "$ENVIRONMENT_ID",
  "initial_events": [
    {"type": "user.message", "content": [{"type": "text", "text": "Run the weekly compliance scan."}]}
  ],
  "schedule": {
    "type": "cron",
    "expression": "0 20 * * 5",
    "timezone": "America/New_York"
  }
}
EOF
```

```python
deployment = client.beta.deployments.create(
    name="Weekly compliance scan",
    agent=agent.id,
    environment_id=environment.id,
    initial_events=[
        {
            "type": "user.message",
            "content": [{"type": "text", "text": "Run the weekly compliance scan."}],
        },
    ],
    schedule={
        "type": "cron",
        "expression": "0 20 * * 5",
        "timezone": "America/New_York",
    },
)
```

响应是一个部署对象（`depl_` ID 前缀）。检查 `schedule.upcoming_runs_at` —— 即下次触发时间 —— 以确认计划按你预期的方式解析：

```json
{
  "id": "depl_01xyz",
  "status": "active",
  "paused_reason": null,
  "schedule": {
    "type": "cron",
    "expression": "0 20 * * 5",
    "timezone": "America/New_York",
    "last_run_at": null,
    "upcoming_runs_at": ["2026-05-09T00:00:00Z", "2026-05-16T00:00:00Z", "2026-05-23T00:00:00Z"]
  }
}
```

部署可能会引入最多 **10 秒的抖动**以分散负载。每个组织最多 **1000 个定时部署**（如需更多请联系 Anthropic 支持）。

### Cron 和时区语义

- **表达式：** 标准 POSIX cron（`minute hour day-of-month month day-of-week`）。
- **时区：** IANA 标识符（例如 `"America/Los_Angeles"`）。
- **夏令时：** 字面墙钟时间匹配 —— `"0 20 * * *"` 在 `America/New_York` 中始终在当地晚上 8:00 触发，无论 EST/EDT。

> ⚠️ **夏令时边界：** 在春季前移日不存在的墙钟时间（如凌晨 2 点）会被**跳过**；在秋季回退日出现两次的时间会**触发两次**。如果不可接受遗漏或重复执行，请将计划安排在凌晨 1-3 点之外，或使用 UTC。

## 部署运行

每次触发尝试 —— 无论成功与否 —— 都会写入一条**部署运行**记录（`drun_` 前缀），因此你可以独立于会话生命周期审计失败。成功的运行携带所创建的 `session_id`；通过事件流（`shared/managed-agents-events.md`）或 webhook（`shared/managed-agents-webhooks.md`）跟踪该会话。失败的运行携带一个 `error`，其 `type` 说明了会话创建被拒绝的原因。

```python
# 某个部署的所有运行记录
for run in client.beta.deployment_runs.list(deployment_id=deployment.id):
    print(run.created_at, run.session_id or run.error.type)

# 仅失败的运行
for run in client.beta.deployment_runs.list(deployment_id=deployment.id, has_error=True):
    print(run.created_at, run.error.type, run.error.message)
```

```typescript
for await (const run of client.beta.deploymentRuns.list({
  deployment_id: deployment.id,
  has_error: true,
})) {
  console.log(run.created_at, run.error?.type, run.error?.message);
}
```

原始 HTTP：`GET /v1/deployment_runs?deployment_id=...&has_error=true`。按 ID 获取单条运行记录：`GET /v1/deployment_runs/{deployment_run_id}`（SDK：`client.beta.deployment_runs.retrieve(run_id)`）—— `deployment_run.*` webhook 事件将运行 ID 作为其 `data.id` 传递。

失败的运行示例：

```json
{
  "type": "deployment_run",
  "id": "drun_01abc124",
  "deployment_id": "depl_01xyz",
  "trigger_context": { "type": "schedule", "scheduled_at": "2026-05-09T00:00:00Z" },
  "session_id": null,
  "error": { "type": "environment_archived", "message": "environment `env_01abc` is archived" },
  "agent": { "type": "agent", "id": "agent_01ghi789", "version": 3 },
  "created_at": "2026-05-09T00:00:01Z"
}
```

错误类型包括 `environment_archived`、`agent_archived`、`vault_not_found`、`session_rate_limited` 和 `service_unavailable`。

每次**定时**运行的结果（started/succeeded/failed）以及每次部署生命周期变更（created/updated/paused/unpaused/archived/deleted）都会作为 webhook 事件发送 —— 参见 `shared/managed-agents-webhooks.md` 中的 `deployment.*` 和 `deployment_run.*` 事件类型 —— 因此你可以无需轮询即可响应。手动运行**不会**发出 `deployment_run.*` webhook 事件。

## 生命周期：暂停 / 恢复 / 归档

| 操作 | SDK | 效果 |
|---|---|---|
| 暂停 | `client.beta.deployments.pause(id)` | 从此停止定时触发。已在运行的会话继续执行。**暂停期间仍允许手动运行。** 设置 `paused_reason: {"type": "manual"}`。 |
| 恢复 | `client.beta.deployments.unpause(id)` | 从下一次计划时间恢复。**错过的触发不会补执行。** 清除 `paused_reason`。 |
| 归档 | `client.beta.deployments.archive(id)` | **终态** —— 计划停止，部署无法再修改。可逆操作请使用暂停。 |

原始 HTTP：`POST /v1/deployments/{deployment_id}/pause`（同理 `/unpause`、`/archive`）。

### 失败行为

- **限流：** 立即记录为 `session_rate_limited` 运行，**不重试** —— 计划仅在下次触发时再次尝试。（会话*内部* API 调用的速率限制由会话自身处理。）
- **其他失败的运行**（例如 `environment_archived`、`vault_not_found`、`service_unavailable`）：运行记录 `error.type` —— 监控运行并修复所引用的资源，或暂停部署。
- **Agent 被归档或删除：** 部署自动**归档**（终态），不再创建新会话。

## 手动运行

`POST /v1/deployments/{deployment_id}/run`（SDK：`client.beta.deployments.run(id)`）立即创建一个会话，并写入一条 `trigger_context.type: "manual"` 的运行记录。用于**在确认计划之前测试部署** —— 注意即使部署处于暂停状态也可使用。