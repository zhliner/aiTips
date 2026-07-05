# HTTP 错误码参考

本文件记录 Claude API 返回的 HTTP 错误码、常见原因及处理方式。关于各语言的错误处理示例，请参阅 `python/` 或 `typescript/` 文件夹。

## 错误码概览

| 状态码 | 错误类型              | 可重试 | 常见原因                         |
| ---- | ----------------------- | --------- | ------------------------------------ |
| 400  | `invalid_request_error` | 否        | 请求格式或参数无效 |
| 401  | `authentication_error`  | 否        | API 密钥无效或缺失           |
| 403  | `permission_error`      | 否        | API 密钥缺少权限             |
| 404  | `not_found_error`       | 否        | 端点或 model ID 无效         |
| 413  | `request_too_large`     | 否        | 请求超出大小限制          |
| 429  | `rate_limit_error`      | 是       | 请求过多                    |
| 500  | `api_error`             | 是       | Anthropic 服务问题              |
| 529  | `overloaded_error`      | 是       | API 暂时过载        |

## 详细错误信息

### 400 Bad Request

**原因：**

- 请求体中的 JSON 格式错误
- 缺少必需参数（`model`、`max_tokens`、`messages`）
- 参数类型无效（例如，应为整数的位置传了字符串）
- messages 数组为空
- messages 未按 user/assistant 交替排列

**错误示例：**

```json
{
  "type": "error",
  "error": {
    "type": "invalid_request_error",
    "message": "messages: roles must alternate between \"user\" and \"assistant\""
  },
  "request_id": "req_011CSHoEeqs5C35K2UUqR7Fy"
}
```

**修复方法：** 发送前验证请求结构。检查：

- `model` 是否为有效的 model ID
- `max_tokens` 是否为正整数
- `messages` 数组是否非空且交替正确

---

### 401 Unauthorized

**原因：**

- 缺少 `x-api-key` header 或 `Authorization` header
- API 密钥格式无效
- API 密钥已被撤销或删除
- OAuth bearer token 通过 `x-api-key` 发送而非 `Authorization: Bearer`
- 同时设置了 `ANTHROPIC_API_KEY` 和 `ANTHROPIC_AUTH_TOKEN` — SDK 会同时发送两个 header，API 拒绝请求

**修复方法：** 设置 `ANTHROPIC_API_KEY`，或运行 `ant auth login` 并保持客户端构造函数为空。对于使用 OAuth token 的原始 HTTP 请求，使用 `Authorization: Bearer <token>`（而非 `x-api-key:`）。

---

### 403 Forbidden

**原因：**

- API 密钥没有对所请求模型的访问权限
- 组织级别的限制
- 尝试在没有 beta 访问权限的情况下访问 beta 功能

**修复方法：** 在 Console 中检查你的 API 密钥权限。你可能需要不同的 API 密钥或申请特定功能的访问权限。

---

### 404 Not Found

**原因：**

- model ID 拼写错误（例如，`claude-sonnet-4.6` 而非 `claude-sonnet-4-6`）
- 使用了已弃用的 model ID
- API 端点无效

**修复方法：** 使用模型文档中的确切 model ID。你可以使用别名（例如 `claude-opus-4-8`）。

---

### 413 Request Too Large

**原因：**

- 请求体超出最大大小
- 输入中的 token 过多
- 图像数据过大

**修复方法：** 减小输入大小——截断对话历史、压缩/调整图像大小，或将大型文档拆分为块。

---

### 400 验证错误

某些 400 错误与参数验证直接相关：

- `max_tokens` 超出模型限制
- `temperature` 值无效（必须为 0.0-1.0）
- 在扩展思考中 `budget_tokens` >= `max_tokens`
- 工具定义 schema 无效

**Fable 5 / Opus 4.8 / 4.7 上的模型特定 400 错误：**

- `temperature`、`top_p`、`top_k` 已被移除——发送其中任何一个都会返回 400。删除该参数；参见 `shared/model-migration.md` → Per-SDK Syntax Reference。
- `thinking: {type: "enabled", budget_tokens: N}` 已被移除——发送它会返回 400。请改用 `thinking: {type: "adaptive"}`。
- **仅限 Fable 5：** 显式的 `thinking: {type: "disabled"}` 返回 400（在 Opus 4.8/4.7 上可以接受）。请完全省略 `thinking` 参数。
- **仅限 Fable 5：** 如果组织设置为零数据保留（ZDR）——或任何低于所需 30 天的保留期——则**所有** Fable 5 请求都会返回 `400 invalid_request_error`，即使 payload 完全有效。在调试请求体之前先检查组织的数据保留配置。

**旧模型（Opus 4.6 及更早版本）上使用扩展思考的常见错误：**

```
# 错误：budget_tokens 必须 < max_tokens
thinking: budget_tokens=10000, max_tokens=1000  → Error!

# 正确
thinking: budget_tokens=10000, max_tokens=16000
```

---

### 429 Rate Limited

**原因：**

- 超出每分钟请求数（RPM）
- 超出每分钟 token 数（TPM）
- 超出每天 token 数（TPD）

**需要检查的 header：**

- `retry-after`：重试前需等待的秒数
- `x-ratelimit-limit-*`：你的限制
- `x-ratelimit-remaining-*`：剩余配额

**修复方法：** Anthropic SDK 会自动使用指数退避重试 429 和 5xx 错误（默认：`max_retries=2`）。有关自定义重试行为，请参阅各语言的错误处理示例。

---

### 500 Internal Server Error

**原因：**

- Anthropic 服务暂时问题
- API 处理中的 Bug

**修复方法：** 使用指数退避重试。如果持续出现，请检查 [status.anthropic.com](https://status.anthropic.com)。

---

### 529 Overloaded

**原因：**

- API 需求量大
- 服务容量已达上限

**修复方法：** 使用指数退避重试。考虑使用不同的模型（Haiku 通常负载较低），分散请求时间，或实现请求队列。

---

## 常见错误及修复

| 错误                         | 错误码            | 修复方法                                                     |
| ------------------------------- | ---------------- | ------------------------------------------------------- |
| 在 Fable 5 / Opus 4.8 / 4.7 上使用 `temperature`/`top_p`/`top_k` | 400 | 删除该参数（参见 `shared/model-migration.md`）  |
| 在 Fable 5 / Opus 4.8 / 4.7 上使用 `budget_tokens` | 400  | 使用 `thinking: {type: "adaptive"}`                      |
| 在 Fable 5 上使用 `thinking: {type: "disabled"}` | 400    | 完全省略 `thinking` 参数（在 Opus 4.8/4.7 上可以接受） |
| 组织设置为 ZDR / 保留期低于 30 天（Fable 5） | 每个请求都返回 400 | 修复组织的数据保留配置——问题不在 payload |
| `budget_tokens` >= `max_tokens`（旧模型） | 400 | 确保 `budget_tokens` < `max_tokens`                  |
| model ID 拼写错误                | 404              | 使用有效的 model ID，如 `claude-opus-4-8`               |
| 第一条消息为 `assistant`    | 400              | 第一条消息必须为 `user`                            |
| 连续相同角色的消息  | 400              | `user` 和 `assistant` 交替排列                        |
| 代码中包含 API 密钥                 | 401（密钥泄露） | 使用环境变量                                |
| 自定义重试需求  | 429/5xx          | SDK 自动重试；使用 `max_retries` 自定义 |

## SDK 中的类型化异常

**始终使用 SDK 的类型化异常类**，而非通过字符串匹配检查错误消息。每个 HTTP 状态码在每个 SDK 中都映射到特定的异常类。

### 各语言的异常类名

| HTTP | Python (`anthropic.*`) / TypeScript (`Anthropic.*`) | Ruby (`Anthropic::Errors::*`) | Java (`com.anthropic.errors.*`) | C# | PHP (`Anthropic\Core\Exceptions\*`) |
|---|---|---|---|---|---|
| 400 | `BadRequestError` | `BadRequestError` | `BadRequestException` | `AnthropicBadRequestException` | `BadRequestException` |
| 401 | `AuthenticationError` | `AuthenticationError` | `UnauthorizedException` | `AnthropicUnauthorizedException` | `AuthenticationException` |
| 403 | `PermissionDeniedError` | `PermissionDeniedError` | `PermissionDeniedException` | `AnthropicForbiddenException` | `PermissionDeniedException` |
| 404 | `NotFoundError` | `NotFoundError` | `NotFoundException` | `AnthropicNotFoundException` | `NotFoundException` |
| 422 | `UnprocessableEntityError` | `UnprocessableEntityError` | `UnprocessableEntityException` | `AnthropicUnprocessableEntityException` | `UnprocessableEntityException` |
| 429 | `RateLimitError` | `RateLimitError` | `RateLimitException` | `AnthropicRateLimitException` | `RateLimitException` |
| ≥500 | `InternalServerError` | `InternalServerError` | `InternalServerException` | `Anthropic5xxException` | `InternalServerException` |
| 网络 | `APIConnectionError` | `APIConnectionError` | `AnthropicIoException` | `AnthropicIOException` | `APIConnectionException` |
| 基础 | `APIError`（两者）；`APIStatusError`（仅 Python） | `APIStatusError` / `APIError` | `AnthropicServiceException` | `AnthropicApiException` | `APIStatusException` / `APIException` |

Ruby 和 PHP 类位于专用的 errors 命名空间中——使用 `Anthropic::Errors::RateLimitError` 和 `Anthropic\Core\Exceptions\RateLimitException`（而非裸 `Anthropic::RateLimitError`）。所有 4xx C# 异常也继承自 `Anthropic4xxException`。

### 从最具体的开始捕获，链式排列

将 `catch`/`except`/`rescue` 子句从最具体的子类排列到基础类，为你不同处理的每个类别设置单独的子句——可重试（429、≥500、网络）vs. 不可重试（4xx）。SDK 为每个状态码定义了不同的类，正是为此目的；单一的宽泛 catch-all 会丢弃这些信息。

```python
try:
    msg = client.messages.create(...)
except anthropic.NotFoundError as e:          # 404 — 例如，错误的 model ID
    ...
except anthropic.RateLimitError as e:         # 429 — 退避并重试
    ...
except anthropic.APIStatusError as e:         # 任何其他非 2xx HTTP 响应
    print(e.status_code, e.message)
except anthropic.APIConnectionError as e:     # 响应前的网络故障
    ...
```

相同的链式结构适用于每个 SDK：TypeScript `instanceof Anthropic.NotFoundError` → `RateLimitError` → `APIConnectionError` → `APIError`（在 `APIError` 之前检查 `APIConnectionError` — 在 TypeScript SDK 中它是 `APIError` 的子类，不同于 Python 中它们是同级关系）；Ruby `rescue Anthropic::Errors::NotFoundError` → `…::RateLimitError` → `…::APIStatusError`；Java `catch (NotFoundException) … catch (RateLimitException) … catch (AnthropicServiceException)`；C# `catch (AnthropicNotFoundException) … catch (AnthropicRateLimitException) … catch (AnthropicApiException)`；PHP `catch (NotFoundException) … catch (RateLimitException) … catch (APIStatusException)`。

### Go — `errors.As` 然后根据状态码分支

Go SDK 对所有非 2xx 响应返回单个 `*anthropic.Error`。使用 `errors.As` 解包，然后根据 `StatusCode` 分支：

```go
_, err := client.Messages.New(ctx, params)
if err != nil {
    var apierr *anthropic.Error
    if errors.As(err, &apierr) {
        switch apierr.StatusCode {
        case 404:
            // 错误的 model ID / 资源
        case 429:
            // 退避并重试
        default:
            // 其他 API 错误 — apierr.StatusCode, apierr.RequestID
        }
    } else {
        // 传输层错误（*url.Error 包装 *net.OpError 等）
    }
}
```

### 错误 `.type` 字段

所有 `APIStatusError` 子类现在都暴露一个 `.type` 属性（Python: `.type`，TypeScript: `.type`，Java: `.errorType()`，Go: `.Type()`，Ruby: `.type`，PHP: `.type`），返回 API 错误类型字符串（例如 `"invalid_request_error"`、`"authentication_error"`、`"rate_limit_error"`、`"overloaded_error"`）。当你需要比 HTTP 状态码更细粒度的编程式错误分类时，使用此属性——例如，区分 `"billing_error"` 和 `"permission_error"`（两者都映射到 403）。

```python
except anthropic.APIStatusError as e:
    if e.type == "rate_limit_error":
        # 处理速率限制
    elif e.type == "overloaded_error":
        # 处理过载
```
