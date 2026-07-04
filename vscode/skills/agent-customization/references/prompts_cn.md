# [Prompts（.prompt.md）](https://code.visualstudio.com/docs/copilot/customization/prompt-files)

在聊天中按需触发的可复用任务模板。单一聚焦任务，支持参数化输入。

## 位置

| 路径 | 作用域 |
|------|--------|
| `.github/prompts/*.prompt.md` | Workspace |
| `<profile>/prompts/*.prompt.md` | 用户配置 |

## Frontmatter

```yaml
---
description: "<推荐>" # 可选，但可提高可发现性
name: "Prompt Name"          # 可选，默认为文件名
argument-hint: "Task..."     # 可选：聊天输入框中显示的提示
agent: "agent"               # 可选：ask、agent、plan 或自定义 agent
model: "GPT-5 (copilot)"     # 可选：选定的模型，或回退数组
tools: [search, web]    # 可选：内置、工具集、MCP（<server>/*）、扩展
---
```

支持模型回退：

```yaml
model: ['GPT-5 (copilot)', 'Claude Sonnet 4.5 (copilot)']
```

## 模板

```markdown
---
description: "Generate test cases for selected code"
agent: "agent"
---
Generate comprehensive test cases for the provided code:
- Include edge cases and error scenarios
- Follow existing test patterns in the codebase
- Use descriptive test names
```

**上下文引用**：使用 Markdown 链接引用文件（`[config](./config.json)`），使用 `#tool:<name>` 引用工具。

## 调用方式

- **聊天**：输入 `/` → 从 prompts 和 skills 中选择
- **命令**：`Chat: Run Prompt...`
- **编辑器**：打开 prompt 文件 → 播放按钮

> Prompts 和 skills 都会作为斜杠命令出现在聊天中。Skills 提供带有捆绑资源的多步骤工作流；prompts 则是单一聚焦任务。

**提示**：使用 `chat.promptFilesRecommendations` 可在开始新聊天时将 prompts 显示为操作项。

## 工具优先级

当 prompt 和自定义 agent 同时定义了工具时：
1. 来自 prompt 文件的工具
2. 来自引用的自定义 agent 的工具
3. 所选 agent 的默认工具

## 适用场景

- 为特定代码生成测试用例
- 从规格说明创建 README
- 使用自定义参数汇总指标
- 一次性生成任务

## 核心原则

1. **单一任务聚焦**：一个 prompt = 一个明确定义的任务
2. **输出示例**：当质量取决于结构时，展示预期格式
3. **复用优于重复**：引用指令文件而非复制内容

## 反模式

- **多任务 prompt**：在一个 prompt 中"创建、测试并部署"
- **模糊的 description**：描述无法帮助用户理解何时使用
- **工具过多**：任务只需要搜索或文件访问时配置了大量工具
