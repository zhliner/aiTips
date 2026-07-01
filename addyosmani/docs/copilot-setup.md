# 在 GitHub Copilot 中使用 agent-skills

## 安装配置（Setup）

### Copilot 指令（Copilot Instructions）

Copilot 支持在仓库中使用 `.github/skills`、`.claude/skills` 或 `.agents/skills` 目录创建智能体技能。

```bash
mkdir -p .github

# 为核心技能创建文件
cat /path/to/agent-skills/skills/test-driven-development/SKILL.md > .github/skills/test-driven-development/SKILL.md
cat /path/to/agent-skills/skills/code-review-and-quality/SKILL.md > .github/skills/code-review-and-quality/SKILL.md
```

更多详情请参阅 [为 GitHub Copilot 创建智能体技能](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/create-skills)。

### 智能体角色（Agent Personas）（*.agent.md）

Copilot 支持专业化的智能体角色。使用 agent-skills 的智能体：

> **重要提示：** GitHub Copilot 要求自定义智能体文件命名为 `*.agent.md`。
> 命名为 `*.md` 的文件会被 Copilot 静默忽略。
> 详情请参阅 [VS Code 自定义智能体文档](https://code.visualstudio.com/docs/copilot/customization/custom-agents#_custom-agent-file-structure)。

```bash
# 创建智能体目录并复制智能体定义
mkdir -p .github/agents
cp /path/to/agent-skills/agents/code-reviewer.md .github/agents/code-reviewer.agent.md
cp /path/to/agent-skills/agents/test-engineer.md .github/agents/test-engineer.agent.md
cp /path/to/agent-skills/agents/security-auditor.md .github/agents/security-auditor.agent.md
```

在 Copilot Chat 中调用智能体：
- `@code-reviewer Review this PR`
- `@test-engineer Analyze test coverage for this module`
- `@security-auditor Check this endpoint for vulnerabilities`

### 自定义指令（用户级）（Custom Instructions）

对于你想在所有仓库中使用的技能：

1. 打开 VS Code → 设置 → GitHub Copilot → 自定义指令
2. 添加你最常用的技能摘要

## 推荐配置（Recommended Configuration）

### .github/copilot-instructions.md

GitHub Copilot 通过 `.github/copilot-instructions.md` 支持项目级指令。

```markdown
# 项目编码规范

## 测试
- 先写测试再写代码（TDD）
- 对于 bug：先写一个失败的测试，然后修复（Prove-It 模式）
- 测试层级：单元 > 集成 > 端到端（使用能捕获行为的最低层级）
- 每次修改后运行 `npm test`

## 代码质量
- 从五个维度审查：正确性、可读性、架构、安全性、性能
- 每个 PR 必须通过：lint、类型检查、测试、构建
- 代码或版本控制中不得包含密钥

## 实现
- 以小型、可验证的增量构建
- 每个增量：实现 → 测试 → 验证 → 提交
- 永远不要将格式化修改与行为修改混在一起

## 边界
- 始终：在提交前运行测试、验证用户输入
- 先询问：数据库 schema 变更、新依赖
- 绝不：提交密钥、删除失败的测试、跳过验证
```

### 专业化智能体（Specialized Agents）

在 Copilot Chat 中使用智能体进行针对性的审查工作流。

## 使用技巧（Usage Tips）

1. **保持指令简洁** — Copilot 指令在聚焦时效果最好。总结关键规则，而非包含完整的技能文件。
2. **使用智能体进行审查** — code-reviewer、test-engineer 和 security-auditor 智能体专为 Copilot 的智能体模型设计。
3. **在对话中引用** — 在处理特定阶段时，将相关技能内容粘贴到 Copilot Chat 中作为上下文。
4. **结合 PR 审查** — 设置 Copilot 使用 code-reviewer 智能体角色审查 PR。
