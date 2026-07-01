# agent-skills 快速入门（Getting Started with agent-skills）

agent-skills 适用于任何接受 Markdown 指令的 AI 编码智能体。本指南介绍通用方法。工具特定的配置请参阅专门的指南。

## 技能如何工作（How Skills Work）

每个技能是一个 Markdown 文件（`SKILL.md`），描述特定的工程工作流。当加载到智能体上下文中时，智能体会遵循该工作流 — 包括验证步骤、需要避免的反模式和退出条件。

**技能不是参考文档。** 它们是智能体遵循的逐步流程。

## 快速开始（适用于任何智能体）（Quick Start - Any Agent）

### 1. 克隆仓库

```bash
git clone https://github.com/addyosmani/agent-skills.git
```

### 2. 选择技能

浏览 `skills/` 目录。每个子目录包含一个 `SKILL.md`，其中有：
- **何时使用** — 表明该技能适用的触发条件
- **流程** — 逐步工作流
- **验证** — 如何确认工作已完成
- **常见合理化借口** — 智能体可能用来跳过步骤的借口
- **红旗** — 技能被违反的迹象

### 3. 将技能加载到智能体

将相关 `SKILL.md` 内容复制到智能体的系统提示词、规则文件或对话中。最常见的方式：

**系统提示词：** 在会话开始时粘贴技能内容。

**规则文件：** 将技能内容添加到项目的规则文件（CLAUDE.md、.cursorrules 等）。

**对话：** 在给出指令时引用技能："按照 test-driven-development 流程进行这次修改。"

### 4. 使用元技能进行发现

先加载 `using-agent-skills` 技能。它包含一个流程图，将任务类型映射到相应的技能。

## 推荐配置（Recommended Setup）

### 最小配置（从这里开始）（Minimal - Start here）

在规则文件中加载三个核心技能：

1. **spec-driven-development** — 用于定义要构建什么
2. **test-driven-development** — 用于证明它能工作
3. **code-review-and-quality** — 用于合并前验证质量

这三个覆盖了 AI 辅助开发中最关键的质量缺口。

### 完整生命周期（Full Lifecycle）

要获得全面覆盖，按阶段加载技能：

```
启动项目：  spec-driven-development → planning-and-task-breakdown
开发期间：  incremental-implementation + test-driven-development
合并之前：  code-review-and-quality + security-and-hardening
部署之前：  shipping-and-launch
```

### 上下文感知加载（Context-Aware Loading）

不要一次加载所有技能 — 这浪费上下文。加载与当前任务相关的技能：

- 在做 UI？加载 `frontend-ui-engineering`
- 在调试？加载 `debugging-and-error-recovery`
- 在搭建 CI？加载 `ci-cd-and-automation`

## 技能结构（Skill Anatomy）

每个技能遵循相同的结构：

```
YAML frontmatter（名称、描述）
├── 概述（Overview）— 这个技能做什么
├── 何时使用（When to Use）— 触发条件
├── 核心流程（Core Process）— 逐步工作流
├── 示例（Examples）— 代码示例和模式
├── 常见合理化借口（Common Rationalizations）— 借口和驳斥
├── 红旗（Red Flags）— 技能被违反的迹象
└── 验证（Verification）— 退出条件清单
```

完整规范请参阅 [skill-anatomy.md](skill-anatomy.md)。

## 使用智能体（Using Agents）

`agents/` 目录包含预配置的智能体角色：

| 智能体 | 用途 |
|--------|------|
| `code-reviewer.md` | 五维度代码审查 |
| `test-engineer.md` | 测试策略和编写 |
| `security-auditor.md` | 漏洞检测 |
| `web-performance-auditor.md` | Core Web Vitals 和性能审计（通过 `/webperf`） |

在需要专业审查时加载智能体定义。例如，要求你的编码智能体"使用 code-reviewer 智能体角色审查这次变更"并提供智能体定义。

## 使用命令（Using Commands）

`.claude/commands/` 目录包含 Claude Code 的斜杠命令：

| 命令 | 调用的技能 |
|------|-----------|
| `/spec` | spec-driven-development |
| `/plan` | planning-and-task-breakdown |
| `/build` | incremental-implementation + test-driven-development |
| `/build auto` | planning-and-task-breakdown → incremental-implementation + test-driven-development（整个计划，一次审批） |
| `/test` | test-driven-development |
| `/review` | code-review-and-quality |
| `/code-simplify` | code-simplification |
| `/ship` | shipping-and-launch |
| `/webperf` | web-performance-auditor（专业智能体，仅用于 Web 应用） |

> **注意：** 作为 Claude Code 插件安装时，你可能会看到类似
> _"Default commands/ folder is ignored because the manifest sets 'commands'"_ 的警告。
> 这是预期行为。根目录的 `commands/` 属于 Antigravity CLI，
> 与 `.claude/commands/` 刻意分开。所有 Claude Code 斜杠
> 命令从 `.claude/commands/` 正确加载；该警告仅为提示性质。

## 使用参考资料（Using References）

`references/` 目录包含补充清单：

| 参考资料 | 配合使用 |
|----------|---------|
| `testing-patterns.md` | test-driven-development |
| `performance-checklist.md` | performance-optimization |
| `security-checklist.md` | security-and-hardening |
| `accessibility-checklist.md` | frontend-ui-engineering |

当需要技能之外的详细模式时加载参考资料。

## 规格说明和任务产物（Spec and task artifacts）

`/spec` 和 `/plan` 命令会创建工作产物（`SPEC.md`、`tasks/plan.md`、`tasks/todo.md`）。在工作进行中将它们视为**活文档**：

- 在开发期间将它们保留在版本控制中，以便人类和智能体有共同的事实来源。
- 当范围或决策变更时更新它们。
- 如果你的仓库不需要长期保留这些文件，在合并前删除它们或将文件夹添加到 `.gitignore` — 工作流不要求它们是永久的。

## 使用技巧（Tips）

1. **对任何非平凡工作从 spec-driven-development 开始**
2. **编写代码时始终加载 test-driven-development**
3. **不要跳过验证步骤** — 它们才是关键
4. **有选择地加载技能** — 更多上下文不一定更好
5. **使用智能体进行审查** — 不同视角捕获不同问题
