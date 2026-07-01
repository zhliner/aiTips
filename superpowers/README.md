# Superpowers

Superpowers 是一套完整的软件开发方法论，专为你的编程智能体设计，建立在一组可组合的技能和一些确保你的智能体使用它们的初始指令之上。

## 我们正在招聘！

我们正在招聘一位全职人员来帮助 Superpowers 的社区和代码工作。
你可以在 https://primeradiant.com/jobs/superpowers-community-engineer/ 阅读职位描述。
如果你觉得身边有人合适，请务必推荐给我们。

## 快速开始

赋予你的智能体 Superpowers：[Claude Code](#claude-code)、[Antigravity](#antigravity)、[Codex App](#codex-app)、[Codex CLI](#codex-cli)、[Cursor](#cursor)、[Factory Droid](#factory-droid)、[GitHub Copilot CLI](#github-copilot-cli)、[Kimi Code](#kimi-code)、[OpenCode](#opencode)、[Pi](#pi)。

## 工作原理

从你启动编程智能体的那一刻就开始了。当它看到你在构建某样东西时，它**不会**直接跳进去写代码。而是退一步，先问你真正想做的是什么。

一旦从对话中梳理出规范，它会以足够短、便于你阅读和消化的片段向你展示。

在你批准设计后，你的智能体会制定一个实施计划，这个计划清晰到足以让一位"品味差、缺乏判断力、没有项目背景、讨厌测试"的热情初级工程师也能遵循。它强调真正的 red/green TDD、YAGNI（你不会需要它的）和 DRY。

接下来，一旦你说了"开始"，它会启动一个**子智能体驱动开发**流程，让智能体完成每个工程任务，检查审查它们的工作，然后继续前进。你的智能体不偏离你一起制定的计划、自主工作一两个小时的情况并不少见。

还有很多其他内容，但这就是系统的核心。而且因为技能是自动触发的，你不需要做任何特殊的事情。你的编程智能体就是拥有了 Superpowers。

## 商业服务

如果你在企业中使用 Superpowers，并且可以从商业支持、额外工具或支出管理中受益，请随时通过 sales@primeradiant.com 联系我们。

## 安装

安装方式因 harness 而异。如果你使用多个 harness，请为每个单独安装 Superpowers。

### Claude Code

Superpowers 可通过[官方 Claude 插件市场](https://claude.com/plugins/superpowers)获取。

#### 官方市场

- 从 Anthropic 官方市场安装插件：

  ```bash
  /plugin install superpowers@claude-plugins-official
  ```

#### Superpowers 市场

Superpowers 市场提供 Superpowers 及一些其他相关的 Claude Code 插件。

- 注册市场：

  ```bash
  /plugin marketplace add obra/superpowers-marketplace
  ```

- 从此市场安装插件：

  ```bash
  /plugin install superpowers@superpowers-marketplace
  ```

### Antigravity

从此仓库以插件形式安装 Superpowers：

```bash
agy plugin install https://github.com/obra/superpowers
```

Antigravity 运行插件的 session-start hook，因此 Superpowers 从第一条消息起就处于激活状态。使用相同命令重新安装即可更新。

### Codex App

Superpowers 可通过[官方 Codex 插件市场](https://github.com/openai/plugins)获取。

- 在 Codex 应用中，点击侧边栏的 Plugins。
- 你应该会在 Coding 部分看到 `Superpowers`。
- 点击 Superpowers 旁边的 `+` 并按照提示操作。

### Codex CLI

Superpowers 可通过[官方 Codex 插件市场](https://github.com/openai/plugins)获取。

- 打开插件搜索界面：

  ```bash
  /plugins
  ```

- 搜索 Superpowers：

  ```bash
  superpowers
  ```

- 选择 `Install Plugin`。

### Cursor

- 在 Cursor Agent 聊天中，从市场安装：

  ```text
  /add-plugin superpowers
  ```

- 或在插件市场中搜索 "superpowers"。

### Factory Droid

- 注册市场：

  ```bash
  droid plugin marketplace add https://github.com/obra/superpowers
  ```

- 安装插件：

  ```bash
  droid plugin install superpowers@superpowers
  ```

### GitHub Copilot CLI

- 注册市场：

  ```bash
  copilot plugin marketplace add obra/superpowers-marketplace
  ```

- 安装插件：

  ```bash
  copilot plugin install superpowers@superpowers-marketplace
  ```

### Kimi Code

Superpowers 可在 Kimi Code 的插件市场中获取。

- 打开 Kimi Code 的插件管理器：

  ```text
  /plugins
  ```

- 前往 `Marketplace` > `Superpowers` 并安装。

- 或直接从此仓库安装：

  ```text
  /plugins install https://github.com/obra/superpowers
  ```

- 详细文档：[docs/README.kimi.md](docs/README.kimi.md)

### OpenCode

OpenCode 使用自己的插件安装方式；即使你已在其他 harness 中使用 Superpowers，也请单独安装。

- 告诉 OpenCode：

  ```
  Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.opencode/INSTALL.md
  ```

- 详细文档：[docs/README.opencode.md](docs/README.opencode.md)

### Pi

从此仓库以 Pi 包形式安装 Superpowers：

```bash
pi install git:github.com/obra/superpowers
```

对于本地开发，以临时包的形式加载此 checkout 运行 Pi：

```bash
pi -e /path/to/superpowers
```

Pi 包加载 Superpowers 技能和一个小型扩展，该扩展在会话启动时和 compaction 后注入 `using-superpowers` bootstrap。Pi 拥有原生技能支持，因此不需要兼容性的 `Skill` 工具。子智能体和任务列表工具仍然是可选的 Pi 伴侣包。

## 基本工作流

1. **brainstorming** - 在编写代码之前激活。通过提问细化粗略想法，探索替代方案，分块展示设计以供验证。保存设计文档。

2. **using-git-worktrees** - 在设计批准后激活。在新分支上创建隔离工作空间，运行项目设置，验证干净的测试基线。

3. **writing-plans** - 在有批准的设计后激活。将工作分解为可一口吃下的任务（每个 2-5 分钟）。每个任务都有确切的文件路径、完整的代码和验证步骤。

4. **subagent-driven-development** 或 **executing-plans** - 有计划后激活。为每个任务分派一个新的子智能体，包含两阶段审查（规范合规性，然后是代码质量），或以批次方式执行并设有人工检查点。

5. **test-driven-development** - 在实施过程中激活。强制执行 RED-GREEN-REFACTOR：编写失败的测试，看它失败，编写最少代码，看它通过，提交。删除在测试之前编写的代码。

6. **requesting-code-review** - 在任务之间激活。对照计划进行审查，按严重程度报告问题。严重问题会阻止进展。

7. **finishing-a-development-branch** - 在任务完成时激活。验证测试，提供选项（merge/PR/keep/discard），清理 worktree。

**智能体在执行任何任务之前都会检查相关技能。** 强制性工作流，而非建议。

## 内部结构

### 技能库

**测试**
- **test-driven-development** - RED-GREEN-REFACTOR 循环（包含测试反模式参考）

**调试**
- **systematic-debugging** - 4 阶段根因流程（包含根因追踪、纵深防御、基于条件的等待技术）
- **verification-before-completion** - 确保问题确实已修复

**协作**
- **brainstorming** - 苏格拉底式设计细化
- **writing-plans** - 详细的实施计划
- **executing-plans** - 带检查点的批量执行
- **dispatching-parallel-agents** - 并行子智能体工作流
- **requesting-code-review** - 提交前审查清单
- **receiving-code-review** - 响应审查反馈
- **using-git-worktrees** - 并行开发分支
- **finishing-a-development-branch** - 合并/PR 决策工作流
- **subagent-driven-development** - 带两阶段审查的快速迭代（规范合规性，然后是代码质量）

**Meta**
- **writing-skills** - 按照最佳实践创建新技能（包含测试方法论）
- **using-superpowers** - 技能系统介绍

## 理念

- **测试驱动开发** - 始终先写测试
- **系统性优先于临时性** - 流程优先于猜测
- **降低复杂度** - 简洁是首要目标
- **证据优先于声明** - 在声称成功之前先验证

阅读[原始发布公告](https://blog.fsck.com/2025/10/09/superpowers/)。

## 贡献

以下是 Superpowers 的一般贡献流程。请注意，我们通常不接受新技能的贡献，并且对技能的任何更新都必须在所有我们支持的编程智能体上正常工作。

1. Fork 仓库
2. 切换到 'dev' 分支
3. 为你所做的工作创建分支
4. 遵循 `writing-skills` 技能来创建和测试新的及修改后的技能
5. 提交 PR，确保填写完整的 pull request 模板

技能行为测试使用来自 [superpowers-evals](https://github.com/prime-radiant-inc/superpowers-evals/) 的 drill 评估 harness，克隆到 `evals/`——参见 `evals/README.md` 进行设置。插件基础设施测试位于 `tests/`，通过相应的 `run-*.sh` 或 `npm test` 运行。

完整指南参见 `skills/writing-skills/SKILL.md`。

## 更新

Superpowers 的更新方式在一定程度上取决于编程智能体，但通常是自动的。

## 许可证

MIT License - 详情见 LICENSE 文件

## 可视化伴侣遥测

由于技能和插件不向创建者提供任何反馈，我们不知道有多少人在使用 Superpowers。默认情况下，brainstorming 的可选可视化伴侣功能中的 Prime Radiant 标志会从我们的网站加载。其中包含正在使用的 Superpowers 版本。它**不**包含有关你的项目、提示或编程智能体的任何详细信息。我们看不到你的点击行为或你正在构建的任何内容。这帮助我们对有多少人在使用 Superpowers 以及他们使用的是哪个版本有一个大致了解。这是 100% 可选的。要禁用此功能，请将环境变量 `SUPERPOWERS_DISABLE_TELEMETRY` 设置为任何真值。Superpowers 也会遵循 Claude Code 的 `DISABLE_TELEMETRY` 和 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 退出选项。

## 社区

Superpowers 由 [Jesse Vincent](https://blog.fsck.com) 和 [Prime Radiant](https://primeradiant.com) 的其他成员共同构建。

- **Discord**：[加入我们](https://discord.gg/35wsABTejz) 获取社区支持、提问和分享你使用 Superpowers 构建的项目
- **Issues**：https://github.com/obra/superpowers/issues
- **发布公告**：[注册](https://primeradiant.com/superpowers/) 以获取新版本通知
