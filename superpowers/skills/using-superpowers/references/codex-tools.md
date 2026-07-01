## Subagent 派发需要 multi-agent 支持

添加到你的 Codex 配置（`~/.codex/config.toml`）：

```toml
[features]
multi_agent = true
```

这将为 `dispatching-parallel-agents` 和 `subagent-driven-development` 等 skill 启用 `spawn_agent`、`wait_agent` 和 `close_agent`。使用 subagent-driven-development 时，应在 implementer 和 reviewer subagent 完成所有工作后始终关闭它们。

## 环境检测

创建 worktree 或完成分支的 skill 应在继续之前使用只读 git 命令检测其环境：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → 已在链接的 worktree 中（跳过创建）
- `BRANCH` 为空 → detached HEAD（无法从沙箱分支/推送/发起 PR）

参见 `using-git-worktrees` 步骤 0 和 `finishing-a-development-branch` 步骤 1，了解各 skill 如何使用这些信号。

## Codex App 完成方式

当沙箱阻止分支/推送操作（外部管理的 worktree 中的 detached HEAD）时，agent 提交所有工作并通知用户使用 App 的原生控件：

- **"创建分支"**——命名分支，然后通过 App UI 提交/推送/发起 PR
- **"移交到本地"**——将工作转移到用户的本地检出

agent 仍然可以运行测试、暂存文件，并输出建议的分支名称、提交信息和 PR 描述供用户复制。
