# Managed Agents — Webhooks

Anthropic 可以在 Managed Agents 资源状态变更时向你配置的 HTTPS 端点发送 POST 请求 —— 作为保持 SSE 流或轮询的替代方案。负载**精简**（仅包含事件类型 + 资源 ID）；收到后，请获取资源以获取当前状态。每次投递都经过 HMAC 签名。

> **方向很重要。** 本页介绍的是 *Anthropic → 你* 的关于会话/Vault 状态的通知。它**不**涉及 *第三方 → 你* 用于*触发*会话的 webhook（例如调用 `sessions.create()` 的 GitHub push 处理器）—— 那是你侧的普通应用代码，没有 Anthropic 特定的报文格式。

---

## 注册端点（仅限 Console）

Console → **Manage → Webhooks**。目前没有编程式的端点管理 API。同一页面支持密钥轮换。

| 字段 | 约束 |
|---|---|
| URL | HTTPS 443 端口， publicly resolvable 主机名 |
| 事件类型 | 按 `data.type` 订阅 —— 你只会收到已订阅的类型（加上测试事件） |
| 签名密钥 | `whsec_` 前缀，32 字节，**创建时仅显示一次** —— 请妥善保存 |

---

## 验证签名

每次投递都经过 HMAC 签名。**请使用 SDK 的 `client.beta.webhooks.unwrap()`** —— 它验证签名、拒绝超过约 5 分钟前的负载，并返回解析后的事件。它从 `ANTHROPIC_WEBHOOK_SIGNING_KEY` 读取 `whsec_` 密钥。

```python
import anthropic
from flask import Flask, request

client = anthropic.Anthropic()  # 从环境变量读取 ANTHROPIC_WEBHOOK_SIGNING_KEY
app = Flask(__name__)


@app.route("/webhook", methods=["POST"])
def webhook():
    try:
        event = client.beta.webhooks.unwrap(
            request.get_data(as_text=True),
            headers=dict(request.headers),
        )
    except Exception:
        return "invalid signature", 400

    if event.id in seen_event_ids:  # 去重重试 —— id 是 per-event 而非 per-delivery
        return "", 204
    seen_event_ids.add(event.id)

    match event.data.type:
        case "session.status_idled":
            session = client.beta.sessions.retrieve(event.data.id)
            notify_user(session)
        case "vault_credential.refresh_failed":
            alert_oncall(event.data.id)

    return "", 204
```

将**原始请求体**传递给 `unwrap()` —— 会重新序列化 JSON 的框架（Express `.json()`、Flask `.get_json()`）会改变字节并破坏 MAC。对于其他语言，请在 SDK 仓库中查找 `beta.webhooks.unwrap` 绑定（`shared/live-sources.md`）；不要自行实现验证。

---

## 负载信封

```json
{
  "type": "event",
  "id": "event_01ABC...",
  "created_at": "2026-03-18T14:05:22Z",
  "data": {
    "type": "session.status_idled",
    "id": "session_01XYZ...",
    "organization_id": "8a3d2f1e-...",
    "workspace_id": "c7b0e4d9-..."
  }
}
```

根据 `data.type` 分支处理，按 `data.id` 获取资源，返回任何 **2xx** 状态码以确认。`created_at` 是*状态转换*发生的时间，而非 webhook 触发的时间。

---

## 支持的 `data.type` 值

| `data.type` | 触发时机 |
|---|---|
| `session.status_scheduled` | 会话已创建并准备好接受事件 |
| `session.status_run_started` | Agent 执行启动（每次转换到 `running`） |
| `session.status_idled` | Agent 等待输入（工具审批、自定义工具结果或下一条消息） |
| `session.status_terminated` | 会话遇到终态错误 |
| `session.thread_created` | 多 Agent：协调器打开了一个新的子 Agent 线程 |
| `session.thread_idled` | 多 Agent：子 Agent 线程正在等待输入 |
| `session.outcome_evaluation_ended` | 结果评估器完成了一次迭代 |
| `vault.archived` | Vault 被归档 |
| `vault.created` | Vault 被创建 |
| `vault.deleted` | Vault 被删除 |
| `vault_credential.archived` | Vault 凭据被归档 |
| `vault_credential.created` | Vault 凭据被创建 |
| `vault_credential.deleted` | Vault 凭据被删除 |
| `vault_credential.refresh_failed` | MCP OAuth Vault 凭据刷新失败 |
| `agent.created` | Agent 被创建 |
| `agent.updated` | 新的 Agent 版本已发布。不创建新版本的更新**不会**触发此事件。 |
| `agent.archived` | Agent 被归档 |
| `agent.deleted` | Agent 被永久删除 —— 没有可获取的对象；将事件本身视为最终状态 |
| `deployment.created` | 定时部署被创建 |
| `deployment.updated` | 部署属性变更（例如计划被编辑） |
| `deployment.paused` | 部署被暂停 —— 通过请求，或在定时运行遇到**不可恢复**错误（Agent 已归档、环境缺失）时自动暂停。可恢复的失败（包括限流）**不会**自动暂停。 |
| `deployment.unpaused` | 部署恢复；计划继续执行 |
| `deployment.archived` | 部署被归档 —— 直接归档，或因 Agent 归档/删除导致 |
| `deployment.deleted` | 部署被永久删除 —— 没有可获取的对象；将事件本身视为最终状态 |
| `deployment_run.started` | **定时**运行已启动。手动运行**不会**发出 `deployment_run.*` 事件。 |
| `deployment_run.succeeded` | 定时运行成功创建了会话。与运行的 `.started` 事件具有相同的 `data.id`（运行 ID）—— 获取部署运行以获取其 `session_id`，然后订阅会话事件以跟踪工作。 |
| `deployment_run.failed` | 定时运行未能创建会话。与运行的 `.started` 事件具有相同的 `data.id` —— 获取部署运行以获取 `error.type` / `error.message`。 |

> 这些是 **webhook** 的 `data.type` 值 —— 与 SSE 事件类型（`session.status_idle`、`span.outcome_evaluation_end` 等，见 `shared/managed-agents-events.md`）是不同的命名空间。不要在 webhook 处理器中复用 SSE 常量。

---

## 投递行为与注意事项

- **无顺序保证。** `session.status_idled` 可能在 `session.outcome_evaluation_ended` 之前到达，即使评估先完成。如果顺序重要，请按信封中的 `created_at` 排序。
- **重试携带相同的 `event.id`。** 非 2xx 响应至少重试一次。按 `event.id` 去重。
- **3xx 视为失败。** 不跟随重定向 —— 如果端点迁移，请在 Console 中更新 URL。
- **自动禁用：** 约 20 次连续投递失败后自动禁用，或当主机名解析到私有 IP 或返回重定向时立即禁用。在 Console 中手动重新启用。
- **精简负载是有意设计。** 不要期望 webhook 负载中包含 `stop_reason`、`outcome_evaluations`、凭据密钥等 —— 请获取资源。