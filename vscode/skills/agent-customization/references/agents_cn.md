# [自定义 Agents（.agent.md）](https://code.visualstudio.com/docs/copilot/customization/custom-agents)

具有特定工具、指令和行为的自定义角色。用于具有基于角色的工具限制的编排工作流。

## 位置

| 路径 | 作用域 |
|------|--------|
| `.github/agents/*.agent.md` | Workspace |
| `<profile>/agents/*.agent.md` | 用户配置 |

## Frontmatter

```yaml
---
description: "<必填>"    # 用于 agent 选择器和 subagent 发现
name: "Agent Name"           # 可选，默认为文件名
tools: [search, web]         # 可选：别名、MCP（<server>/*）、扩展工具
model: "Claude Sonnet 4"     # 可选，使用选择器默认值；支持数组形式的回退
argument-hint: "Task..."     # 可选，输入引导
agents: [agent1, agent2]     # 可选，按名称限制允许的 subagents（省略 = 全部，[] = 无）
user-invocable: true         # 可选，在 agent 选择器中显示（默认：true）
disable-model-invocation: false  # 可选，阻止 subagent 调用（默认：false）
handoffs: [...]              # 可选，转移到其他 agents
hooks:                       # 可选，此 agent 生命周期事件的内联 hooks
  PreToolUse:
    - type: command
      command: "./scripts/validate.sh"
  PostToolUse:
    - type: command
      command: "./scripts/format.sh"
---
```

### 调用控制

| 属性 | 默认值 | 效果 |
|------|--------|------|
| `user-invocable: false` | `true` | 从 agent 选择器中隐藏，仅可作为 subagent 访问 |
| `disable-model-invocation: true` | `false` | 阻止其他 agents 将其作为 subagent 调用 |

### 模型回退

```yaml
model: ['Claude Sonnet 4.5 (copilot)', 'GPT-5 (copilot)']  # 使用第一个可用的模型
```

## 工具

来源：内置别名、特定工具、MCP 服务器（`<server>/*`）、扩展工具。

**特殊**：`[]` = 无工具，省略 = 默认值。正文引用：`#tool:<name>`

### 工具别名

| 别名 | 用途 |
|------|------|
| `execute` | 运行 shell 命令 |
| `read` | 读取文件内容 |
| `edit` | 编辑文件 |
| `search` | 搜索文件或文本 |
| `agent` | 调用自定义 agents 作为 subagents |
| `web` | 获取 URL 和网页搜索 |
| `todo` | 管理任务列表 |

### 常见模式

```yaml
tools: [read, search]             # 只读研究
tools: [myserver/*]               # 仅 MCP 服务器
tools: [read, edit, search]       # 无终端访问
tools: []                         # 仅对话
```

要发现可用工具，请检查当前工具列表或在正文中使用 `#tool:` 语法引用特定工具。

## 模板

```markdown
---
description: "{Use when... 用于 subagent 发现的触发短语}"
tools: [{最小工具别名集}]
user-invocable: false
---
You are a specialist at {specific task}. Your job is to {clear purpose}.

## Constraints
- DO NOT {this agent 绝不应做的事情}
- DO NOT {另一个限制}
- ONLY {this agent 唯一要做的事}

## Approach
1. {this agent 工作方式的第一步}
2. {第二步}
3. {第三步}

## Output Format
{this agent 应返回的确切内容}
```

## 调用方式

- **手动**：聊天中的 agent 选择器
- **Subagent**：父 agent 基于 `description` 匹配进行委派（当 `infer` 允许时）

## 核心原则

1. **单一角色**：每个 agent 一个聚焦职责的角色
2. **最小工具**：只包含角色所需的工具——过多工具会分散注意力
3. **清晰边界**：定义 agent 不应做什么
4. **关键词丰富的 description**：包含触发词，以便父 agent 知道何时委派

## 反模式

- **瑞士军刀式 agents**：工具过多，试图做所有事情
- **模糊的 description**："A helpful agent" 无法指导委派——要具体
- **角色混乱**：description 与正文角色不匹配
- **循环转移**：A → B → A 没有进展标准

## 内联 Hooks

自定义 agents 支持 frontmatter 中的内联 `hooks`。这些 hooks 在 agent 生命周期节点执行 shell 命令，且仅作用于该 agent。格式与独立 hook 文件一致（参见 [hooks 参考](../hooks.md)）。

### 支持的事件

`SessionStart`、`UserPromptSubmit`、`PreToolUse`、`PostToolUse`、`PreCompact`、`SubagentStart`、`SubagentStop`、`Stop`

### 示例

```yaml
---
description: "Secure code reviewer that blocks dangerous commands"
tools: [read, search, execute]
hooks:
  PreToolUse:
    - type: command
      command: "./scripts/block-dangerous-cmds.sh"
      timeout: 10
  PostToolUse:
    - type: command
      command: "./scripts/auto-lint.sh"
---
```

每个 hook 命令支持：`type`（必须为 `command`）、`command`、平台覆盖（`windows`、`linux`、`osx`）、`cwd`、`env`、`timeout`。
