# Token 计数

使用 `count_tokens` 端点（`POST /v1/messages/count_tokens`）获取 Claude 模型的准确 token 计数。Token 计数是**模型特定的**——传入与推理时相同的模型 ID。

**不要使用 `tiktoken`。** 它是 OpenAI 的分词器。在典型文本上，它会低估 Claude token 约 15–20%，在代码或非英语输入上低估更多。任何来自 `tiktoken`、`gpt-tokenizer` 或类似工具的估算对 Claude 来说都是错误的。

## 对文件或字符串进行计数

```python
from anthropic import Anthropic

client = Anthropic()
resp = client.messages.count_tokens(
    model="claude-opus-4-8",
    messages=[{"role": "user", "content": open("CLAUDE.md").read()}],
)
print(resp.input_tokens)
```

TypeScript：`await client.messages.countTokens({model, messages})` →
`.input_tokens`。其他 SDK 请参见 `{lang}/claude-api/README.md`。

## CLI

```sh
ant messages count-tokens --model claude-opus-4-8 \
  --message '{role: user, content: "@./CLAUDE.md"}' \
  --transform input_tokens -r
```

## 比较文件两个版本间的差异

该端点是无状态的——分别对每个版本进行计数然后相减：

```python
from anthropic import Anthropic
import subprocess

client = Anthropic()
def count(text: str) -> int:
    return client.messages.count_tokens(
        model="claude-opus-4-8",
        messages=[{"role": "user", "content": text}],
    ).input_tokens

before = subprocess.check_output(["git", "show", "HEAD:CLAUDE.md"], text=True)
after = open("CLAUDE.md").read()
print(count(after) - count(before))
```

完整文档：参见 `shared/live-sources.md` 中的 Token Counting 条目。