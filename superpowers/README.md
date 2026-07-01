# Superpowers（超能力）

Superpowers 是一套完整的编码代理软件开发工作流，基于一组可组合的"技能"（skills）以及一些初始指令构建，确保你的代理能够使用这些技能。

## How it works（工作原理）

它从你启动编码代理的那一刻就开始了。当代理发现你要构建某些东西时，它*不会*直接跳进去写代码。相反，它会退后一步，问你真正想要做什么。

一旦它从对话中梳理出规格说明，就会以足够短、可以实际阅读和消化的片段展示给你。

在你确认设计方案后，你的代理会制定一个实现计划，这个计划清晰到一个热情但品味不佳、没有判断力、没有项目背景、又厌恶测试的初级工程师也能照着做。它强调真正的红/绿 TDD、YAGNI（你不会需要它的）和 DRY。

接下来，当你说"开始"后，它会启动一个*子代理驱动开发*流程，让代理逐个完成工程任务，检查和审查他们的工作，然后继续推进。Claude 能够自主工作数小时而不偏离你制定的计划，这并不罕见。

还有更多细节，但这就是系统的核心。而且由于技能会自动触发，你不需要做任何特别的事情。你的编码代理天生就拥有 Superpowers。


## Sponsorship（赞助）

如果 Superpowers 帮助你赚到了钱，并且你愿意的话，我非常感谢你考虑[赞助我的开源工作](https://github.com/sponsors/obra)。

谢谢！

- Jesse


## Installation（安装）

**注意：** 不同平台的安装方式不同。Claude Code 或 Cursor 有内置的插件市场。Codex 和 OpenCode 需要手动设置。


### Claude Code（通过插件市场）

在 Claude Code 中，首先注册市场：

```bash
/plugin marketplace add obra/superpowers-marketplace
```

然后从这个市场安装插件：

```bash
/plugin install superpowers@superpowers-marketplace
```

### Cursor（通过插件市场）

在 Cursor Agent 聊天中，从市场安装：

```text
/plugin-add superpowers
```

### Codex

告诉 Codex：

```
Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.codex/INSTALL.md
```

**详细文档：** [docs/README.codex.md](docs/README.codex.md)

### OpenCode

告诉 OpenCode：

```
Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.opencode/INSTALL.md
```

**详细文档：** [docs/README.opencode.md](docs/README.opencode.md)

### Verify Installation（验证安装）

在你选择的平台中启动一个新会话，并要求触发某个技能的操作（例如，"帮我规划这个功能"或"让我们调试这个问题"）。代理应该会自动调用相关的 superpowers 技能。

## The Basic Workflow（基本工作流）

1. **brainstorming** - 在编写代码之前激活。通过提问完善粗略想法，探索替代方案，分段展示设计方案以供验证。保存设计文档。

2. **using-git-worktrees** - 在设计批准后激活。在新分支上创建隔离工作区，运行项目设置，验证干净的测试基线。

3. **writing-plans** - 在设计批准后激活。将工作拆分为细粒度任务（每个 2-5 分钟）。每个任务都有确切的文件路径、完整代码和验证步骤。

4. **subagent-driven-development** 或 **executing-plans** - 在计划就绪后激活。为每个任务分派新的子代理并进行两阶段审查（规格合规性，然后代码质量），或分批执行并设置人工检查点。

5. **test-driven-development** - 在实现过程中激活。强制执行 RED-GREEN-REFACTOR：编写失败的测试，看它失败，编写最少代码，看它通过，提交。删除在测试之前编写的代码。

6. **requesting-code-review** - 在任务之间激活。对照计划审查，按严重程度报告问题。关键问题会阻止进度。

7. **finishing-a-development-branch** - 在任务完成时激活。验证测试，提供选项（合并/PR/保留/丢弃），清理工作树。

**代理在任何任务之前都会检查相关技能。** 这是强制性工作流，不是建议。

## What's Inside（内容概览）

### Skills Library（技能库）

**Testing（测试）**
- **test-driven-development** - RED-GREEN-REFACTOR 循环（包含测试反模式参考）

**Debugging（调试）**
- **systematic-debugging** - 4 阶段根因分析流程（包含根因追踪、纵深防御、基于条件等待技术）
- **verification-before-completion** - 确保问题真正被修复

**Collaboration（协作）**
- **brainstorming** - 苏格拉底式设计完善
- **writing-plans** - 详细实现计划
- **executing-plans** - 带检查点的分批执行
- **dispatching-parallel-agents** - 并并发子代理工作流
- **requesting-code-review** - 预审查清单
- **receiving-code-review** - 回应反馈
- **using-git-worktrees** - 并行开发分支
- **finishing-a-development-branch** - 合并/PR 决策工作流
- **subagent-driven-development** - 快速迭代，两阶段审查（规格合规性，然后代码质量）

**Meta（元技能）**
- **writing-skills** - 按照最佳实践创建新技能（包含测试方法论）
- **using-superpowers** - 技能系统介绍

## Philosophy（理念）

- **Test-Driven Development** - 始终先写测试
- **Systematic over ad-hoc** - 流程优于猜测
- **Complexity reduction** - 以简洁为首要目标
- **Evidence over claims** - 在宣布成功之前先验证

阅读更多：[Superpowers for Claude Code](https://blog.fsck.com/2025/10/09/superpowers/)

## Contributing（贡献）

技能直接存放在这个仓库中。参与贡献：

1. Fork 这个仓库
2. 为你的技能创建一个分支
3. 按照 `writing-skills` 技能创建和测试新技能
4. 提交 PR

参见 `skills/writing-skills/SKILL.md` 获取完整指南。

## Updating（更新）

更新插件时技能会自动更新：

```bash
/plugin update superpowers
```

## License（许可证）

MIT License - 详见 LICENSE 文件

## Support（支持）

- **Issues**: https://github.com/obra/superpowers/issues
- **Marketplace**: https://github.com/obra/superpowers-marketplace
