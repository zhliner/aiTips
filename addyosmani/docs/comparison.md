<!--
  本文档面向评估本项目的开发者。它不是一个技能，
  不应被加载到智能体的上下文中。它放在 docs/ 目录下，
  以避免进入智能体的工作集。
-->

# agent-skills 与其他项目的对比（How agent-skills compares）

人们经常问 **agent-skills** 与另外两个流行的"编码智能体技能"合集之间的关系：**Superpowers**（Jesse Vincent / obra 出品）和 **Matt Pocock 的 skills**。三者都很优秀，有很多共同基因，都值得学习。本页诚实地梳理了它们的*结构性差异*，帮助你选择最适合自己工作方式的那个 — 或者从多个项目中借鉴。

> **总结** — 它们优化的是不同场景。**agent-skills** 组织*整个产品生命周期*（定义 → 计划 → 构建 → 验证 → 审查 → 发布），配备审查角色和反合理化防线。**Superpowers** 偏向*自主、重推理*的运行模式，使用子智能体和 worktree 隔离。**Matt Pocock 的 skills** 是一套*精练、个人化的 Claude Code 工具箱*，提炼自一位专家的日常实践。没有抽象意义上的"最好" — 取决于你面前的工作。

---

## 一览（At a glance）

| | **agent-skills** | **Superpowers** | **Matt Pocock 的 skills** |
|---|---|---|---|
| **核心理念** | 将完整的高级工程生命周期编码为技能 | 基于可组合技能构建的完整开发*方法论* | 一位专家的 `.claude` 工作流，开源分享 |
| **组织原则** | SDLC **阶段**（定义→计划→构建→验证→审查→发布），配元技能路由 | 纪律执行循环（头脑风暴 → 计划 → 执行） | 精选的命令工具箱 |
| **生命周期覆盖** | 广泛 — 想法精炼、API/UI 设计、安全、性能、CI/CD、废弃、ADR、发布 | 深入核心构建循环（TDD、调试、计划、审查） | 计划 + 构建 + 工具 + 知识管理，有主见 |
| **入口** | 斜杠命令与阶段一一对应（`/spec` `/plan` `/build` `/test` `/review` `/code-simplify` `/ship`，加上 `/webperf`） | 如 `/brainstorming`、`/execute-plan` 等命令 | 如 `/tdd`、`/grill-me`、`/diagnose`、`/grill-with-docs` 等斜杠命令 |
| **工具覆盖** | 多工具：Claude Code、Cursor、Gemini CLI、Antigravity、OpenCode、Windsurf、Copilot | 多工具：Claude Code、Codex、Gemini CLI、OpenCode、Cursor、Copilot CLI、Factory Droid | Claude Code 优先（也可用于 Codex） |
| **独特机制** | 反合理化表格 + 每个技能中的红旗警示；审查**角色**在 `/ship` 中并行扇出；参考清单 | 子智能体驱动开发，两阶段审查；git-worktree 隔离；技能编写技能 | "Grill me"需求拷问；严格的智能体级 TDD；pre-commit/git 护栏 |
| **最适合** | 驱动功能经历每个阶段，每个阶段都有人类检查点 | 长时间、自主、重推理或探索性工作 | TypeScript 风格项目的务实、经实战检验的日常循环 |

*（各博客对这些项目的采用数据引用差异很大；我们选择不列出，以免传播未经验证的数字。）*

---

## 三个项目各自的定位（The three projects, in their own terms）

### Superpowers - obra
基于可组合技能构建的完整软件开发方法论。它押注于**自主性和前置推理**：编码前的苏格拉底式头脑风暴、执行任务并接受两阶段审查（规格合规性 + 代码质量）的全新子智能体，以及确保并行工作隔离的 git worktree。其 TDD 纪律严格 — 会删除过早编写的代码以坚守 RED→GREEN→REFACTOR 路线。如果你想交出一大块工作然后回来看到审查后的结果，这就是为此而设计的形态。

**仓库：** <https://github.com/obra/superpowers>

### Matt Pocock 的 skills - mattpocock
Matt 开源了他日常使用的 `.claude` 目录 — 一套精练的 Claude Code 技能。亮点包括 `/tdd`（在智能体层面强制执行红-绿-重构）和 `/grill-me`（在任何代码之前拷问你的需求）。还涵盖 PRD 编写、issue 拆分、接口设计、架构审查、bug 分诊、pre-commit/git 护栏和知识管理。它以一种最好的方式体现了个人化和有主见：反映了一位优秀工程师实际交付的方式，而非试图成为穷举式框架。

**仓库：** <https://github.com/mattpocock/skills> · 相关：<https://github.com/mattpocock/agent-rules-books>

### agent-skills - 本项目
agent-skills 将**整个产品生命周期**组织为技能，通过元技能（`using-agent-skills`）将任务路由到正确的技能。每个技能都包含**常见合理化借口**表格（智能体跳过步骤的借口，逐一驳斥）和**红旗**警示。斜杠命令与生命周期阶段一一对应，`/ship` 并行扇出审查**角色** — `code-reviewer`、`security-auditor`、`test-engineer`、`web-performance-auditor` — 然后合并为通过/不通过决策。它在每个阶段刻意保留人类检查点，并跨大多数主流智能体工具运行。

---

## 实际对比：Superpowers vs. agent-skills（A real head-to-head）

Om Mishra 进行了一项对照实验 — 相同模型（Sonnet 4.6）、相同仓库、相同 Claude Code 提示词，仅更换技能框架 — 并撰写了报告：

**["Superpowers vs Agent-Skills: Faster Shipping, Safer Reasoning"](https://www.linkedin.com/pulse/superpowers-vs-agent-skills-faster-shipping-safer-reasoning-om-mishra-dzakf/)** - Om Mishra

他的发现，公正地总结：

- **agent-skills** 更快地进入编码阶段（约 8 分钟 vs 约 12 分钟），并执行了**更多验证轮次**（7 轮 vs 5 轮，包括完整测试套件）。更广泛的验证捕获了一个*在当前功能之外*的兼容性问题，而功能特定的测试未能发现。对于该任务，他认为 agent-skills 在**验证深度**上更胜一筹。
- **Superpowers** 投入了更多**前置架构推理**，他仍然将其作为日常首选，用于演进生产系统和没有既定模式可遵循的探索性工作。
- Token 效率基本相同；两者都重新规划了一次。

这是一位开发者的单任务实验，不是基准测试 — 但它是一个有用的、具体的核心权衡说明：**广泛的纪律验证 vs. 重前置推理。** 他自己的结论是最诚实的：根据任务选择工具。

---

## 何时选择哪个（When to pick which）

- **选择 agent-skills**：当你需要**引导式生命周期**，每个阶段有人类检查点，合并前有并行审查/安全/性能检查，覆盖范围延伸到构建循环之外的安全、性能、CI/CD 和发布。它还跨最多的智能体工具运行。
- **选择 Superpowers**：当你想**交出长时间、自主的工作段**然后回来看到审查后的结果，或者工作是探索性/架构性的，受益于更重的前置推理和子智能体隔离。
- **选择 Matt Pocock 的 skills**：当你需要一套**精练、低仪式的日常工具箱** — 特别是需求拷问和严格 TDD 循环 — 用于 TypeScript 风格的 Claude Code 工作流。

你不必排他性地选择，但组合时需谨慎。这些是 Markdown 技能，不是运行时，所以挑选*单个*技能效果很好：在你的主要设置中引入 Matt 的 `grill-me`、Superpowers 的子智能体隔离，或某个特定清单。

不可行的是同时运行两个作为**活跃路由器**。叠加的元技能会争夺命令名（`/tdd` 在两处定义）、竞争路由逻辑、引入不同的 TDD 哲学，导致不可预测的行为而非两者之长。选择一个框架作为主路由器，从其他框架中按需借鉴。

---

## 来源（Sources）

- Superpowers - <https://github.com/obra/superpowers>
- Matt Pocock 的 skills - <https://github.com/mattpocock/skills>
- Om Mishra, *Superpowers vs Agent-Skills* - <https://www.linkedin.com/pulse/superpowers-vs-agent-skills-faster-shipping-safer-reasoning-om-mishra-dzakf/>

*发现关于其他项目的信息不准确？请提 issue 或 PR — 我们宁可公正也不奉承。*
