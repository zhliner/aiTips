## Subagent 派遣需要多 agent 支持

添加到你的 Codex 配置（`~/.codex/config.toml`）：

```toml
[features]
multi_agent = true
```

这将启用 `spawn_agent`、`wait_agent` 和 `close_agent`，供 `dispatching-parallel-agents` 和 `subagent-driven-development` 等 skills 使用。使用 subagent-driven-development 时，你应该始终在 implementer 和 reviewer subagents 完成所有工作后关闭它们。

## 环境检测

创建 worktrees 或完成分支的 skills 应在继续之前使用只读 git 命令检测环境：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → 已在关联 worktree 中（跳过创建）
- `BRANCH` 为空 → detached HEAD（无法从沙箱中 branch/push/PR）

参见 `using-git-worktrees` 步骤 0 和 `finishing-a-development-branch` 步骤 1，了解各 skill 如何使用这些信号。

## Codex App 完成流程

当沙箱阻止 branch/push 操作时（在外部管理的 worktree 中为 detached HEAD），agent 提交所有工作并告知用户使用 App 的原生控件：

- **"Create branch"**——命名分支，然后通过 App UI 进行 commit/push/PR
- **"Hand off to local"**——将工作转移到用户的本地 checkout

Agent 仍然可以运行测试、暂存文件，并输出建议的分支名称、commit 消息和 PR 描述供用户复制。
