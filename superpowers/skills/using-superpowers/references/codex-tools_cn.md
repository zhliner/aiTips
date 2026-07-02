## Subagent 分派需要 multi-agent 支持

在你的 Codex 配置（`~/.codex/config.toml`）中添加以下内容：

```toml
[features]
multi_agent = true
```

这为 `dispatching-parallel-agents` 和 `subagent-driven-development` 等技能启用了 `spawn_agent`、`wait_agent` 和 `close_agent`。使用 subagent-driven-development 时，你应始终在 implementer 和 reviewer subagent 完成所有工作后关闭它们。

## 环境检测

创建 worktree 或完成分支的技能在继续之前，应使用只读 git 命令检测其环境：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → 已在链接 worktree 中（跳过创建）
- `BRANCH` 为空 → detached HEAD（无法在沙箱中分支/推送/创建 PR）

参见 `using-git-worktrees` 的步骤 0 和 `finishing-a-development-branch` 的步骤 1，了解各技能如何使用这些信号。

## Codex App 完成

当沙箱阻止分支/推送操作时（externally managed worktree 中的 detached HEAD），agent 提交所有工作并告知用户使用 App 的原生控件：

- **"Create branch"** —— 命名分支，然后通过 App UI 完成 commit/push/PR
- **"Hand off to local"** —— 将工作转移到用户的本地 checkout

Agent 仍然可以运行测试、暂存文件，并输出建议的分支名称、commit message 和 PR 描述供用户复制。
