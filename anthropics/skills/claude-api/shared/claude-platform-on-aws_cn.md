# Claude Platform on AWS

通过 AWS 基础设施进行 **Anthropic 运营**的 Claude 开发者平台访问——SigV4 身份验证、AWS IAM 访问控制和 AWS Marketplace 计费。由于由 Anthropic 运营，**API 表面与第一方完全一致，当天同步**——关于各功能的例外情况，请参阅 `shared/platform-availability.md`（唯一权威来源；请勿依赖此处的内联例外列表）。Model ID 为裸第一方字符串（`claude-opus-4-8`、`claude-sonnet-5`）——**无 provider 前缀**。

> **与 Amazon Bedrock 不同。** Bedrock 由合作伙伴运营（AWS 运行服务；发布时间表不一，功能子集，`anthropic.` 前缀的 model ID）。Claude Platform on AWS 和 Bedrock 共存；根据你是否需要 AWS 原生 IAM/计费并具备完整的 Anthropic API 一致性（本页）vs. Bedrock 自身的生态系统来选择。

---

## 客户端与安装

| 语言 | 安装 | 客户端 |
|---|---|---|
| Python | `pip install -U "anthropic[aws]"` | `from anthropic import AnthropicAWS` → `AnthropicAWS()` |
| TypeScript | `npm install @anthropic-ai/aws-sdk` | `import AnthropicAws from "@anthropic-ai/aws-sdk"` → `new AnthropicAws()` |
| Go | `go get github.com/anthropics/anthropic-sdk-go` | `import anthropicaws "github.com/anthropics/anthropic-sdk-go/aws"` → `anthropicaws.NewClient(ctx, anthropicaws.ClientConfig{})` |
| C# | `dotnet add package Anthropic.Aws` | `new AnthropicAwsClient()` |
| Java | 参见 `shared/live-sources.md` 中的 SDK 仓库 | 参见 `shared/live-sources.md` 中的 SDK 仓库 |
| Ruby | `gem install anthropic aws-sdk-core` | 参见 `shared/live-sources.md` 中的 SDK 仓库 |
| PHP | `composer require anthropic-ai/sdk aws/aws-sdk-php` | 参见 `shared/live-sources.md` 中的 SDK 仓库 |

构造之后，**使用客户端的方式与 `Anthropic()` 完全相同** — `client.messages.create(...)`、`client.beta.sessions.*` 等，使用裸 model ID。

```python
from anthropic import AnthropicAWS

client = AnthropicAWS()  # region + workspace_id 从环境变量获取；见下文
client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}],
)
```

---

## 必需配置

必须提供两个值（构造函数参数或环境变量）——**两者均无默认回退**：

| 值 | 环境变量 | 说明 |
|---|---|---|
| AWS region | `AWS_REGION` | 必需。与 `AnthropicBedrock` 不同，没有 `us-east-1` 回退。 |
| Workspace ID | `ANTHROPIC_AWS_WORKSPACE_ID` | 必需。将请求路由到你的 Claude 工作区。 |

端点模式：`https://aws-external-anthropic.{region}.api.aws/v1/...`。请求使用服务名 `aws-external-anthropic` 进行 SigV4 签名。

## 身份验证

客户端通过标准优先级链解析 AWS 凭据：显式构造函数参数 → 环境变量（`AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`/`AWS_SESSION_TOKEN`）→ 共享 profile → 假定角色 / 实例元数据。

**短期 API 密钥**也适用于 SigV4 不切实际的场景（例如浏览器、简单脚本）。使用各语言的 token 生成器包生成一个；将其作为 `api_key` 传递给客户端。有效期为请求的时长、底层凭据的过期时间和 **12 小时**中的**最小值**。有关包名和 IAM 详情，请 WebFetch `shared/live-sources.md` 中的 Claude Platform on AWS 页面。

---

## 需要告知用户的信息

- 将其视为第一方：本技能的每个部分均适用且无需更改。**不要**应用 Bedrock 的功能可用性过滤。
- Model ID 为裸格式（`claude-opus-4-8`）。**不要**添加 `anthropic.` 前缀。
- 缺少 region 或 `workspace_id` 会在客户端构造时抛出异常（不会发送请求）。**403** 表示请求已到达服务器——检查 `workspace_id` 是否**错误**或 principal 上缺少 IAM action。参见 `shared/live-sources.md` 中的 IAM actions 参考。
