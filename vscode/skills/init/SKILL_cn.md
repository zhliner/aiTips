---
name: init
description: 为 AI 编码 agent 生成或更新 chat 自定义文件
argument-hint: 可选指定要为 agent 记录的聚焦领域或模式
disable-model-invocation: true
---

此命令的目的是创建或更新 chat 自定义文件
- agent instructions 文件（`.github/copilot-instructions.md` 或 `AGENTS.md`），帮助 AI 编码 agent 理解代码库并立即高效工作
- skill 和自定义 agent，用于自动化常见任务或强制执行代码库中的约定

用户可以选择性地附带参数调用此命令。参数可以是对自定义文件的具体请求，或者对于新项目，可以是项目的描述。当附带参数调用时，专注于与该参数相关的自定义。仅创建或修改 chat 自定义文件。绝不开始处理参数中的任务。

当命令被调用时，立即告知用户你正在探索代码库并着手创建和改进 chat 自定义文件。如果用户提供了参数，还要提及你正在聚焦于该领域或模式。保持输出简洁，并在需要时请求反馈或额外输入。

使用相关 skill `agent-customization` 获取有关不同类型自定义文件的详细信息。
探索代码库以充分了解项目及其约定，然后创建或更新相关的 chat 自定义文件，以帮助 AI 编码 agent 在此代码库中高效工作。

完成后，打印一个表格，列出已添加或修改的 chat 自定义文件，并简要说明为什么该文件对 AI 编码 agent 有用。

## 工作流

1. **发现现有约定**
   搜索：`**/{.github/copilot-instructions.md,AGENT.md,AGENTS.md,CLAUDE.md,.cursorrules,.windsurfrules,.clinerules,.cursor/rules/**,.windsurf/rules/**,.clinerules/**,README.md}`

2. **通过 subagent 探索代码库**，如需要可并行 1-3 个
   找到帮助 AI agent 立即高效工作的关键知识：
   - 构建/测试命令（agent 会自动运行这些命令）
   - 架构决策和组件边界
   - 与常见实践不同的项目特定约定
   - 潜在的陷阱或常见的开发环境问题
   - 体现模式的關鍵文件/目录

   同时盘点现有文档（`docs/**/*.md`、`CONTRIBUTING.md`、`ARCHITECTURE.md` 等），以识别应链接而非重复的主题。

3. **生成或合并**
   - 新文件：优先选择 AGENTS.md 而非 `.github/copilot-instructions.md`。如果用户已有其中某个文件，则更新它而非创建新文件。
   - 已有文件：保留有价值的内容，更新过时的部分，去除重复
   - 遵循 `agent-customization` skill 中的指南：
      1. **链接而非嵌入**原则。不要复制 workspace 中已有的文档，改用 Markdown 链接指向它们。
      2. **默认最小化**：仅包含相关且 agent 不易自行发现的内容。链接到其他文档以获取详细信息。
      3. **简洁且可操作**：每一行都应指导行为

4. **迭代**
   - 就不清晰或不完整的部分请求反馈
   - 如果 workspace 较复杂，建议为特定领域（例如前端、后端、测试）创建单独的 instructions 文件或 skill

确定后，提议接下来要创建的相关 agent 自定义（`/create-(agent|hook|instruction|prompt|skill) …`），解释该自定义及其实际使用方式。

如果会话历史可用，使用 **chronicle** skill 检查过去会话中的摩擦模式——这可以发现仅靠代码库探索无法揭示的项目特定约定或陷阱。向用户提及 `/chronicle improve` 作为随时间迭代改进 instructions 的方式。
