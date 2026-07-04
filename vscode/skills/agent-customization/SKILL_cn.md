---
name: agent-customization
user-invocable: false # 不作为斜杠命令显示，我们有专门的 create-agent、create-instructions、create-hook 提示词来处理
description: '**工作流 SKILL** — 创建、更新、审查、修复或调试 VS Code agent 自定义文件（.instructions.md、.prompt.md、.agent.md、SKILL.md、copilot-instructions.md、AGENTS.md）。适用场景：保存编码偏好；排查 instructions/skills/agents 被忽略或未被调用的问题；配置 applyTo 模式；定义工具限制；创建自定义 agent 模式或专业工作流；打包领域知识；修复 YAML frontmatter 语法。不适用场景：一般编码问题（使用默认 agent）；运行时调试或错误诊断；MCP server 配置（直接使用 MCP 文档）；VS Code 扩展开发。调用：文件系统工具（读取/写入自定义文件）、ask-questions 工具（向用户询问需求）、用于代码库探索的 subagent。单一操作：对于快速的 YAML frontmatter 修复或基于已知模式创建单个文件，直接编辑文件即可 — 无需使用 skill。'
---

# Agent 自定义

## 决策流程

| 原语 | 使用时机 |
|------|----------|
| agent instructions | 始终生效，适用于项目中的所有位置 |
| File Instructions | 通过 `applyTo` 模式显式指定，或通过 `description` 按需触发 |
| MCP | 集成外部系统、API 或数据 |
| Hooks | 在 agent 生命周期节点执行确定性 shell 命令（阻止工具、自动格式化、注入上下文） |
| Custom Agents | 用于上下文隔离的 subagent，或带有工具限制的多阶段工作流 |
| Prompts | 具有参数化输入的单一聚焦任务 |
| Skills | 带有捆绑资源（脚本/模板）的按需工作流 |

## 快速参考

查阅参考文档以获取模板、领域示例、高级 frontmatter 选项、资源组织、反模式和创建清单。如果参考文档不够用，加载每个原语的官方文档链接。

| 类型 | 文件 | 位置 | 参考 |
|------|------|------|------|
| agent instructions | `copilot-instructions.md`、`AGENTS.md` | `.github/` 或根目录 | [链接](./references/agent-instructions.md) |
| File Instructions | `*.instructions.md` | `.github/instructions/` | [链接](./references/instructions.md) |
| Prompts | `*.prompt.md` | `.github/prompts/` | [链接](./references/prompts.md) |
| Hooks | `*.json` | `.github/hooks/` | [链接](./references/hooks.md) |
| Custom Agents | `*.agent.md` | `.github/agents/` | [链接](./references/agents.md) |
| Skills | `SKILL.md` | `.github/skills/<name>/`、`.agents/skills/<name>/`、`.claude/skills/<name>/` | [链接](./references/skills.md) |

**用户级别**：`{{VSCODE_USER_PROMPTS_FOLDER}}/`（*.prompt.md、*.instructions.md、*.agent.md；不含 skills）
自定义内容随用户的设置同步漫游

## 创建流程

如果需要探索或验证代码库中的模式，使用只读 subagent。如果 ask-questions 工具可用，使用它来向用户询问并澄清需求。

创建任何自定义文件时请遵循以下步骤。

### 1. 确定范围

询问用户希望将自定义放在哪里：
- **Workspace**：用于项目特定的、团队共享的自定义 → `.github/` 文件夹
- **用户配置**：用于个人跨 workspace 的自定义 → `{{VSCODE_USER_PROMPTS_FOLDER}}/`

### 2. 选择合适的原语

使用上方的决策流程，根据用户需求选择合适的文件类型。

### 3. 创建文件

在适当的路径直接创建文件：
- 使用每个参考文件中的位置表
- 根据需要包含必需的 frontmatter
- 按照模板添加正文内容

### 4. 验证

创建后：
- 确认文件位于正确位置
- 验证 frontmatter 语法（`---` 标记之间的 YAML）
- 检查 `description` 是否存在且有实际意义

## 边界情况

**Instructions 还是 Skill？** 这适用于*大多数*工作，还是*特定*任务？大多数 → Instructions。特定 → Skill。

**Skill 还是 Prompt？** 两者都作为斜杠命令出现在 chat 中（输入 `/`）。多步骤工作流且带有捆绑资源 → Skill。具有输入的单一聚焦任务 → Prompt。

**Skill 还是 Custom Agent？** 所有步骤的能力相同 → Skill。需要上下文隔离（subagent 返回单一输出）或每个阶段需要不同的工具限制 → Custom Agent。

**Hooks 还是 Instructions？** Instructions *引导* agent 行为（非确定性）。Hooks 通过 shell 命令在 `PreToolUse` 或 `PostToolUse` 等生命周期事件处*强制执行*行为 — 它们可以阻止操作、要求审批或确定性地运行格式化程序。Hooks 可以定义在独立的 `.json` 文件中（参见 [hooks 参考](./references/hooks.md)），也可以通过 `hooks` 属性内联在自定义 agent 的 frontmatter 中（参见 [agents 参考](./references/agents.md)）。

## 常见陷阱

**Description 是发现入口。** `description` 字段是 agent 决定是否加载 skill、instruction 或 agent 的依据。如果触发短语不在 description 中，agent 就找不到它。使用"Use when..."模式配合具体关键词。

**YAML frontmatter 的静默失败。** 值中未转义的冒号、使用制表符而非空格、`name` 与文件夹名称不匹配 — 都会导致没有错误消息的静默失败。始终对包含冒号的描述使用引号：`description: "Use when: doing X"`。

**`applyTo: "**"` 消耗上下文。** 这意味着"对每个文件请求始终包含" — 它会在每次交互时将 instruction 加载到上下文窗口中，即使不相关。使用特定的 glob 模式（`**/*.py`、`src/api/**`），除非 instruction 确实适用于所有文件。
