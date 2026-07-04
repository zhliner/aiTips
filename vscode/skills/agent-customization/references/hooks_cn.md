# [Hooks（.json）](https://code.visualstudio.com/docs/copilot/customization/hooks)

Agent 会话的确定性生命周期自动化。使用 hooks 来执行政策、自动化验证和注入运行时上下文。

## 位置

| 路径 | 作用域 |
|------|--------|
| `.github/hooks/*.json` | Workspace（团队共享） |
| `.claude/settings.local.json` | Workspace 本地（不提交） |
| `.claude/settings.json` | Workspace |
| `~/.claude/settings.json` | 用户配置 |

所有已配置位置的 hooks 都会被收集并执行；workspace 和用户 hooks 不会相互覆盖。

## Hook 事件

| 事件 | 触发条件 |
|------|----------|
| `SessionStart` | 新 agent 会话的第一条提示 |
| `UserPromptSubmit` | 用户提交提示 |
| `PreToolUse` | 工具调用之前 |
| `PostToolUse` | 工具成功调用之后 |
| `PreCompact` | 上下文压缩之前 |
| `SubagentStart` | Subagent 启动 |
| `SubagentStop` | Subagent 结束 |
| `Stop` | Agent 会话结束 |

## 配置格式

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "type": "command",
        "command": "./scripts/validate-tool.sh",
        "timeout": 15
      }
    ]
  }
}
```

每个 hook 命令支持：
- `type`（必须为 `command`）
- `command`（默认）
- `windows`、`linux`、`osx`（平台覆盖）
- `cwd`、`env`、`timeout`

## 输入 / 输出契约

Hooks 通过 stdin 接收 JSON，并可通过 stdout 返回 JSON。

- 常见输出：`continue`、`stopReason`、`systemMessage`
- `PreToolUse` 权限从 `hookSpecificOutput.permissionDecision` 读取（`allow` | `ask` | `deny`）
- `PostToolUse` 输出可通过 `decision: block` 阻止后续处理

`PreToolUse` 输出示例：

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "ask",
    "permissionDecisionReason": "Needs user confirmation"
  }
}
```

退出码：
- `0` 成功
- `2` 阻塞错误
- 其他值产生非阻塞警告

## Hooks 与其他自定义方式的对比

| 原语 | 行为 |
|------|------|
| Instructions / Prompts / Skills / Agents | 指导（非确定性） |
| Hooks | 运行时强制执行和确定性自动化 |

当行为必须得到保证时使用 hooks（例如：阻止危险命令、强制验证、自动注入上下文）。

## 核心原则

1. 保持 hooks 小巧且可审计
2. 验证和清理 hook 输入
3. 避免在脚本中硬编码密钥
4. 团队策略优先使用 workspace hooks，个人自动化使用用户 hooks

## 反模式

- 运行长时间阻塞正常流程的 hooks
- 在普通指令即可满足需求时使用 hooks
- 允许 agents 在没有审批控制的情况下编辑 hook 脚本
