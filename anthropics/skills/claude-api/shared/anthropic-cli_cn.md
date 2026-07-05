# Anthropic CLI (`ant`)

`ant` CLI 将每个 Claude API 资源暴露为 shell 子命令。与 `curl` 相比：请求体通过类型化的 flag 或管道输入的 YAML 构建，而非手写 JSON，`@path` 将文件内容内联到任何字符串字段中，`--transform` 使用 GJSON 路径提取字段（无需 `jq`），列表端点自动分页（使用 `--max-items N` 限制总结果数；`--limit` 仅设置服务端分页大小），`beta:` 前缀自动设置正确的 `anthropic-beta` header。

## 何时使用 CLI vs SDK

**CLI 用于控制面，SDK 用于数据面。** Agent 和环境是相对静态的资源，你用 `ant` 定义、配置和调试它们——将 YAML 提交到你的仓库，从 CI 应用，从终端检查。会话是动态的，由你的应用程序通过 SDK 驱动——为每个任务创建，流式传输事件，响应工具调用，集成到你的产品中。两者调用的是同一个 API；区别在于调用发生在哪里，而非能做什么。

| | 控制面 → `ant` | 数据面 → SDK |
|---|---|---|
| 资源 | agents, environments, skills, vaults, files | sessions, events |
| 频率 | 每次部署 / 临时性 | 每个任务 / 每轮 |
| 存在于 | 仓库中的 `*.yaml` + CI + 终端 | 应用程序代码 |
| 典型调用 | `create < agent.yaml`, `update --version N`, `list`, `retrieve`, `archive`, `--debug` | `sessions.create()`, `events.stream()`, `events.send()` |

## 安装与认证

```sh
# macOS
brew install anthropics/tap/ant
xattr -d com.apple.quarantine "$(brew --prefix)/bin/ant"

# Linux / WSL — 从 github.com/anthropics/anthropic-cli/releases 选择对应的发布版
curl -fsSL "https://github.com/anthropics/anthropic-cli/releases/download/v${VERSION}/ant_${VERSION}_$(uname -s | tr A-Z a-z)_$(uname -m | sed -e s/x86_64/amd64/ -e s/aarch64/arm64/).tar.gz" \
  | sudo tar -xz -C /usr/local/bin ant

# 或从源码编译（Go 1.22+）
go install github.com/anthropics/anthropic-cli/cmd/ant@latest
```

**认证** — CLI 按照与 SDK 相同的方式解析凭据（优先匹配）：显式 flag → `ANTHROPIC_API_KEY` → `ANTHROPIC_AUTH_TOKEN` → `ANTHROPIC_PROFILE` 选择的或激活的 profile → Workload Identity Federation 环境变量 → 磁盘上的默认 profile。使用 `ANTHROPIC_BASE_URL` 或 `--base-url` 覆盖主机地址。

- **API 密钥**：在环境变量中设置 `ANTHROPIC_API_KEY`。
- **OAuth profile**（无需管理静态密钥）：`ant auth login` 打开浏览器，交换为短期 token，并在 `$ANTHROPIC_CONFIG_DIR` 下存储一个 profile（默认为 Linux/macOS 上的 `~/.config/anthropic/`，Windows 上的 `%APPDATA%\Anthropic`——`configs/<profile>.json` 存储设置，`credentials/<profile>.json` 存储 token）。后续的 `ant`（和 SDK）调用会自动使用它——登录之后，裸 `Anthropic()` 客户端即可工作，但直接读取 `ANTHROPIC_API_KEY` 的脚本则不行。Claude Code 和 Claude Agent SDK 遵循相同的 profile 解析规则。`ant auth status` 显示哪个凭据来源和 profile 生效（仅报告状态——不要将其退出码作为健康检查来编写脚本）；`ant auth logout` 清除激活的 profile（`--all` 清除所有 profile）。在没有浏览器的远程主机上，`ant auth login --no-browser` 打印授权 URL 并在终端中接受返回的 code。
- **非交互式工作负载**（CI、服务器、容器）：交互式登录用于在你自己的机器上进行开发——请改用 Workload Identity Federation（参见 `shared/live-sources.md` 中的身份验证文档）。

> **最常见的认证陷阱：** profile 仅在没有设置 API 密钥时才会被使用。一个过期的 `ANTHROPIC_API_KEY` 导出会静默覆盖所有 profile——请求会发送到该密钥所属的任何组织/工作区。`ant auth status` 显示哪个来源生效；在使用 profile 之前取消设置该密钥（或按命令：`env -u ANTHROPIC_API_KEY ant …`）。真正地**取消设置**它——空的 `ANTHROPIC_API_KEY=""` 仍然会占据其优先级位置并以空密钥进行认证。同样的遮蔽效应也适用于 Claude Code：在 `ant auth login` 之后，Claude Code 可能会警告 profile 和其自身的 `/login` 凭据之间存在认证冲突——保留其中一个（使用 profile 并在 Claude Code 中 `/logout`，或 `ant auth logout` 以保留 Claude Code 自身的登录）。

**命名 profile** — 交互式登录 token 绑定到单个组织+工作区，API 仅显示属于该工作区的资源。如果你创建的 agent、会话或文件"消失"了，通常的原因是 token 所属的工作区与创建它的工作区不同（`ant auth status` 显示激活的工作区）。多工作区工作意味着每个工作区一个 profile：

```sh
ant auth login --profile <name>                  # 如果 profile 不存在则创建；在浏览器中选择组织/工作区
ant auth login --profile <name> --workspace-id wrkspc_01...   # 直接绑定，跳过选择器
ant profile activate <name>                      # 切换默认 profile
ant --profile <name> models list                 # 一次性使用；等效：ANTHROPIC_PROFILE=<name> ant models list
ant profile list                                 # 查看
ant profile set workspace_id wrkspc_01... --profile <name>    # 编辑配置键（workspace_id、base_url、organization_id 等）
```

`ant profile set` 编辑现有 profile 的配置——它从不创建新 profile，也**不会**重新绑定已颁发的凭据；在该 profile 下重新运行 `ant auth login` 以为新目标生成 token。将 `ANTHROPIC_PROFILE` 指向不存在的 profile 会报错，而非回退。Refresh token 最终会硬过期（不会因使用而延长）——当之前正常工作的 profile 开始认证失败时，在调试其他问题之前先重新运行 `ant auth login`。

**权限范围（Scopes）** — profile 的 OAuth scope 集在登录时请求（`--scope`）并持久化到 profile 上（`scope` 也是 `profile set` 的配置键；与其他配置编辑一样，更改它需要重新执行 `ant auth login` 才能生效）。特权 scope——例如 `org:admin` 用于组织管理端点——**不在**默认 scope 集中：显式传递你需要的完整集合（`ant auth login --profile admin --scope "... org:admin"`），服务器仅在你的角色确实拥有该权限时才会授予特权 scope。由于 scope 集随 profile 生成的每个 token 一起传递，将特权工作放在专用 profile 上（`admin` vs `default`），并在非特权 profile 上进行日常推理，使用 `--profile`/`ANTHROPIC_PROFILE` 切换。使用 `ant auth login --help` 查看当前的 scope 列表，使用 `ant auth status` 查看激活 token 携带的 scope。

要将激活的凭据传递给子进程或原始 HTTP 脚本：

```sh
# 裸 access token — 用于 curl 的 Authorization header
curl https://api.anthropic.com/v1/messages \
  -H "Authorization: Bearer $(ant auth print-credentials --access-token)" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: oauth-2025-04-20" \
  -H "content-type: application/json" \
  -d '{"model": "claude-opus-4-8", "max_tokens": 1024, "messages": [{"role": "user", "content": "Hello"}]}'

# .env 格式 — 设置 ANTHROPIC_AUTH_TOKEN（如果 profile 有设置，还会设置 ANTHROPIC_BASE_URL）。
# 输出为裸 KEY=value（无 `export`），因此使用 `set -a` 自动为子进程导出：
set -a; eval "$(ant auth print-credentials --env)"; set +a
python my_script.py   # SDK 会使用 ANTHROPIC_AUTH_TOKEN
```

OAuth token 放在 `Authorization: Bearer` 上（而非 `x-api-key:`）**加上 `anthropic-beta: oauth-2025-04-20` header** — 将原始 curl/httpx 脚本从 API 密钥转换只需更改 header，而非交换密钥。Beta header 的要求取决于端点（某些端点恰好不需要它；`/v1/messages` 则需要）——始终发送它，这样切换端点时请求不会中断。Token 是短期的，通过环境变量传递时不会自动刷新，因此对于长时间运行的脚本，在过期前重新运行 `print-credentials`（`print-credentials` 本身会在需要时刷新 token）。如果同时设置了 `ANTHROPIC_API_KEY` 和 `ANTHROPIC_AUTH_TOKEN`，SDK 会同时发送两者，API 会拒绝请求——在 `eval` `--env` 输出之前取消设置 `ANTHROPIC_API_KEY`。

**易错点：** `ant auth print-credentials` **不带任何 flag** 会打印完整的凭据 JSON，而非裸 token——将其放入 `Authorization` header 会导致空响应或 HTTP/2 协议错误。始终使用 `--access-token` 获取 header（它始终读取命名/激活的 profile；设置的 `ANTHROPIC_API_KEY` 不会覆盖凭据打印）。

## 命令结构

```
ant <resource>[:<subresource>] <action> [flags]
```

Beta 资源（agents、sessions、environments、deployments、skills、vaults、memory stores）位于 `beta:` 下——CLI 自动发送正确的 `anthropic-beta` header，因此不要自己传递，除非使用 `--beta <header>` 覆盖。对于自托管环境，`ant beta:worker poll/run` 和 `ant beta:environments:work stats/stop` 驱动和监控工作队列——参见 `shared/managed-agents-self-hosted-sandboxes.md`。

```sh
ant models list
ant messages create --model claude-opus-4-8 --max-tokens 1024 --message '{role: user, content: "Hello"}'
ant beta:agents retrieve --agent-id agent_01...
ant beta:sessions:events list --session-id session_01...
```

`ant --help` 列出资源；在任何子命令后追加 `--help` 查看其 flag。

## 全局 flag

| Flag | 用途 |
| --- | --- |
| `--format` | `auto`（默认：TTY 时为 pretty，管道时为 compact）、`json`、`jsonl`、`yaml`、`pretty`、`raw`、`explore`（交互式 TUI） |
| `--transform` | 应用于响应的 GJSON 路径（在列表端点上为逐项应用）。当 `--format raw` 时不应用。 |
| `-r`, `--raw-output` | 如果转换后的结果是字符串，则不带引号打印（jq 语义）。与 `--transform` 配合使用以捕获标量值。 |
| `--max-items` | 限制自动分页列表端点返回的总结果数（与 `--limit` 不同，后者是服务端分页大小）。 |
| `--format-error` / `--transform-error` | 与 `--format`/`--transform` 相同，应用于错误响应。`-r` 不适用于错误路径——使用 `--format-error yaml` 获取不带引号的错误标量。 |
| `--base-url` | 覆盖 API 主机 |
| `--debug` | 将完整的 HTTP 请求 + 响应打印到 stderr（API 密钥会被脱敏） |

## 输出 — `--transform` + `--format`

`--transform` 接受一个 [GJSON 路径](https://github.com/tidwall/gjson/blob/master/SYNTAX.md)。在列表端点上，它**逐项**运行，而非在整个信封上运行。

```sh
ant beta:agents list --transform '{id,name,model}' --format jsonl
```

**为 shell 使用提取标量值：** 将 `--transform` 与 `-r`（`--raw-output` — 不带引号打印字符串，jq 风格）配合使用：

```sh
AGENT_ID=$(ant beta:agents create --name "My Agent" --model '{id: claude-sonnet-5}' \
  --transform id -r)
```

## 输入 — flag、stdin、`@file`

**Flag** — 标量字段直接映射。结构化字段接受宽松 YAML 语法（不带引号的键）或严格 JSON。可重复的 flag 构建数组（每个 `--tool`、`--event`、`--message` 追加一个元素）：

```sh
ant beta:agents create \
  --name "Research Agent" \
  --model '{id: claude-opus-4-8}' \
  --tool '{type: agent_toolset_20260401}' \
  --tool '{type: custom, name: search_docs, input_schema: {type: object, properties: {query: {type: string}}}}'
```

**Stdin** — 管道输入完整的 JSON 或 YAML 体。与 flag 合并；冲突时 flag 优先（对于数组字段，任何 flag **完全替换** stdin 数组——而非追加）。引用 heredoc 分隔符（`<<'YAML'`）以禁用 body 内的 shell 展开：

```sh
ant beta:agents create <<'YAML'
name: Research Agent
model: claude-opus-4-8
system: |
  You are a research assistant. Cite sources for every claim.
tools:
  - type: agent_toolset_20260401
YAML
```

**`@file` 引用** — 将文件内容内联到任何字符串值的字段中。在结构化 flag 值内部，用引号包裹路径。二进制文件会自动 base64 编码；使用 `@file://`（文本）或 `@data://`（base64）强制指定。将字面量前导 `@` 转义为 `\@`。

```sh
ant beta:agents create --name "Researcher" --model '{id: claude-sonnet-5}' --system @./prompts/researcher.txt

ant messages create --model claude-opus-4-8 --max-tokens 1024 \
  --message '{role: user, content: [
    {type: document, source: {type: base64, media_type: application/pdf, data: "@./scan.pdf"}},
    {type: text, text: "Extract the text from this scanned document."}
  ]}' \
  --transform 'content.0.text' -r
```

原生接受文件路径的 flag（例如 `beta:files upload` 上的 `--file`）直接接受裸路径，无需 `@`。

## 版本控制的 Managed Agents 资源

这是定义 agent 和环境的推荐流程——将 YAML 提交到你的仓库，通过 `create`（首次）/ `update`（后续）同步。字段参考请参见 `shared/managed-agents-core.md`。

```yaml
# summarizer.agent.yaml
name: Summarizer
model: claude-sonnet-5
system: |
  You are a helpful assistant that writes concise summaries.
tools:
  - type: agent_toolset_20260401
```

```sh
# 创建（一次性）— 获取 ID
AGENT_ID=$(ant beta:agents create < summarizer.agent.yaml --transform id -r)

# 更新（CI）— 需要 ID + 当前版本（乐观锁）
ant beta:agents update --agent-id "$AGENT_ID" --version 1 < summarizer.agent.yaml
```

环境也采用相同模式（`ant beta:environments create|update < env.yaml`），然后使用两个 ID 启动会话：

```sh
ant beta:sessions create --agent "$AGENT_ID" --environment-id "$ENV_ID" --title "Task"
ant beta:sessions:events send --session-id "$SID" \
  --event '{type: user.message, content: [{type: text, text: "Summarize X"}]}'
ant beta:sessions:events list --session-id "$SID" --transform 'content.0.text' -r
ant beta:sessions:events stream --session-id "$SID"   # 实时事件流
```

### 交互式会话循环（先流式传输再发送）

`ant beta:sessions:events stream` 仅传递在流打开*之后*发出的事件——因此请在发送启动消息**之前**打开流，以避免错过早期事件。使用进程替换将流保持在文件描述符上，发送，然后读取：

```sh
exec {stream}< <(ant beta:sessions:events stream --session-id "$SID" \
  --transform '{type,text:content.#(type=="text").text,err:error.message}' --format yaml)

ant beta:sessions:events send --session-id "$SID" > /dev/null <<'YAML'
events:
  - type: user.message
    content:
      - type: text
        text: Summarize the repo README
YAML

type=
while IFS= read -r -u "$stream" line; do
  case "$line" in
    type:\ session.status_idle) break ;;
    type:\ session.error)
      IFS= read -r -u "$stream" next || next=
      case "$next" in err:\ *) msg=${next#err: } ;; *) msg=unknown ;; esac
      printf '\n[Error: %s]\n' "$msg"; break ;;
    type:\ *) type=${line#type: } ;;
    text:*)
      [[ $type == agent.message ]] || continue
      val=${line#text: }
      case "$val" in '|-'|'|') ;; *) printf '%s' "$val" ;; esac ;;
    \ \ *)
      if [[ $type == agent.message ]]; then printf '%s\n' "${line#  }"; fi ;;
  esac
done
exec {stream}<&-
```

这适用于交互式探索和演示。对于需要响应 `agent.tool_use` / `agent.custom_tool_use` 事件、在断开后重连、或根据 `events.list` 去重的应用程序代码，请使用 SDK——参见 `shared/managed-agents-client-patterns.md`。

## 脚本模式

列表端点上的 `--transform id -r` 每行输出一个裸 ID——与 `xargs` 组合使用，或使用 `--max-items N` 限制结果集而无需通过 `head` 管道：

```sh
FIRST=$(ant beta:agents list --transform id -r --max-items 1)
ant beta:agents:versions list --agent-id "$FIRST" --transform '{version,created_at}' --format jsonl
```

错误处理与成功路径对称（注意：`-r` 不适用于错误输出——此处使用 `--format-error yaml` 获取不带引号的标量）：

```sh
ant beta:agents retrieve --agent-id bogus --transform-error error.message --format-error yaml 2>&1
```

Shell 补全：`ant @completion {zsh|bash|fish|powershell}`。

有关完整的、始终最新的参考（包括每个端点的 flag），请 WebFetch `shared/live-sources.md` 中的 **Anthropic CLI** URL。
