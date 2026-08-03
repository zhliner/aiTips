---
name: using-agent-skills
description: 发现并调用 agent skills。在开始会话，或需要确定哪个 skill 适用于当前任务时使用。这是管理所有其他 skill 如何被发现和调用的元 skill（meta-skill）。
---

# 使用 Agent Skills

## 概述

Agent Skills 是一组按开发阶段组织的工程工作流 skills。每个 skill 都编码了资深工程师会遵循的特定流程。这个元 skill（meta-skill）可以帮助你为当前任务发现并应用正确的 skill，包括在描述相互重叠的 skill 之间做选择。

## Skill 发现

当任务到来时，先识别其所属的开发阶段，再应用相应的 skill：

```
任务到来
    │
    ├── 正在处理 OpenCode 自己的配置/agents/skills/plugins/MCP？ ──→ customize-opencode
    ├── 想要创建、编辑或基准测试一个 skill？ ─────────────────────→ skill-creator
    │
    ├── 还不知道自己想要什么？ ───────────────────────────────────→ interview-me
    ├── 用户随口提出了一个完整功能/概念想要构建？ ─────────────→ brainstorming
    ├── 有一个大致概念，需要做一些变体？ ───────────────────────→ idea-refine
    ├── 新项目 / 功能 / 变更，需要正式规格说明？ ───────────────→ spec-driven-development
    │
    ├── 已经有规格说明，需要计划和任务拆分？ ───────────────────→ planning-and-task-breakdown
    ├── 已经有规格说明，需要一份自包含的计划文档？ ─────────────→ writing-plans
    │
    ├── 准备执行一份已写好的计划？
    │   ├── 需要先隔离工作区 ───────────────────────────────────→ using-git-worktrees
    │   ├── 需要单独会话，并在检查点前由人类复审 ─────────────→ executing-plans
    │   ├── 当前会话中，每个任务都派发一个新的 subagent ───────→ subagent-driven-development
    │   └── 2 个及以上相互独立的任务，没有共享状态 ───────────→ dispatching-parallel-agents
    │
    ├── 正在实现代码？ ───────────────────────────────────────────→ incremental-implementation
    │   ├── UI 工作？ ───────────────────────────────────────────→ frontend-ui-engineering
    │   ├── API 工作？ ───────────────────────────────────────────→ api-and-interface-design
    │   ├── 需要更好的上下文？ ─────────────────────────────────→ context-engineering
    │   ├── 需要文档验证后的代码？ ─────────────────────────────→ source-driven-development
    │   └── 风险高 / 对代码不熟悉？ ───────────────────────────→ doubt-driven-development
    ├── 正在编写 / 运行测试？ ───────────────────────────────────→ test-driven-development
    │   └── 基于浏览器的测试？ ─────────────────────────────────→ browser-testing-with-devtools
    ├── 某些东西坏了？ ───────────────────────────────────────────→ debugging-and-error-recovery
    ├── 即将声称“已完成 / 已修复 / 已通过”？ ─────────────────→ verification-before-completion
    │
    ├── 正在审查代码？ ───────────────────────────────────────────→ code-review-and-quality
    │   ├── 想在合并前先让专门的 reviewer subagent 审查？ ─────→ requesting-code-review
    │   ├── 过于复杂？ ─────────────────────────────────────────→ code-simplification
    │   ├── 有安全问题？ ───────────────────────────────────────→ security-and-hardening
    │   └── 有性能问题？ ───────────────────────────────────────→ performance-optimization
    │
    ├── 实现已完成，测试已通过，准备决定如何集成？ ─────────────→ finishing-a-development-branch
    ├── 正在提交 / 创建分支？ ───────────────────────────────────→ git-workflow-and-versioning
    ├── 正在处理 CI/CD 流水线工作？ ─────────────────────────────→ ci-cd-and-automation
    ├── 正在弃用 / 迁移系统？ ───────────────────────────────────→ deprecation-and-migration
    ├── 正在编写文档 / ADR？ ───────────────────────────────────→ documentation-and-adrs
    ├── 正在添加日志 / 指标 / 告警？ ───────────────────────────→ observability-and-instrumentation
    └── 正在部署 / 发布？ ─────────────────────────────────────→ shipping-and-launch
```

## 选择重叠技能时的判断

有些 skill 看起来很相似。请根据这些区别来选，而不要靠猜：

- brainstorming 与 interview-me + idea-refine + spec-driven-development：brainstorming 是一个轻量级的单轮对话流程（理解 → 每次只问一个问题 → 提出设计 → 获得批准），适用于大多数正常规模的功能请求。只有当需求足够大，需要独立的探索、细化和正式规格三个阶段，或者用户明确按名称调用其中某个 skill 时，才应该使用那三个 skill 的链式流程。
- planning-and-task-breakdown 与 writing-plans：两者都能把 spec 变成计划。planning-and-task-breakdown 是更深入的过程，涉及依赖图、并行化分析、tasks/plan.md 和 tasks/todo.md，适用于会跨多个 agent 或会话的工作。writing-plans 则更轻量：为一个对代码库背景一无所知的工程师，写一份自包含的计划文档。若范围或并行性不清楚，使用前者；若只是想先把计划写下来再开始编码，使用后者。
- executing-plans、subagent-driven-development 和 dispatching-parallel-agents：这三个都会通过 subagent 执行预先写好的工作，主要差异在于会话/隔离模型。executing-plans 会在单独会话中运行，并在检查点前由人类复审。subagent-driven-development 会在当前会话中，为每个任务派发一个新的 implementer subagent，并在每一步后进行复审。dispatching-parallel-agents 是任何 2 个及以上相互独立任务的通用原语，无论是否与计划有关。when the work needs workspace isolated from what you're currently looking at. using-git-worktrees 可与这三者中的任何一个配合使用。
- code-review-and-quality 与 requesting-code-review：code-review-and-quality 是复审过程和检查清单本身；requesting-code-review 是委派机制——用精心准备的上下文把 reviewer subagent 调起来，这样你的会话历史不会混入审查。用 requesting-code-review 来调用 code-review-and-quality（或任何审查流程）。
- verification-before-completion 与各个 skill 自己的验证步骤：verification-before-completion 是适用于所有场景的最终关卡——无论你使用的 skill 是否已经有自己的验证步骤，只要你准备声称某事“已完成”“已修复”或“已通过”，都必须先运行它。
- customize-opencode 和 skill-creator 完全不属于开发生命周期。只有当任务是关于 OpenCode 自己的配置（opencode.json、agents、skills、plugins、MCP servers、权限）或技能系统本身时，才使用它们；绝不要把它们用于目标代码库的应用代码。

## 核心操作行为

这些行为在所有 skills 中始终适用，而且是不可妥协的。

### 1. 暴露假设

在实施任何非平凡工作之前，明确陈述你的假设：

```
我正在做出的假设：
1. [关于需求的假设]
2. [关于架构的假设]
3. [关于范围的假设]
→ 请现在纠正我，否则我会按这些假设继续。
```

不要默默地把模糊需求补全。最常见的失败模式是做出错误假设，然后不加检查地继续。尽早暴露不确定性——这比返工便宜得多。

### 2. 主动处理困惑

当你遇到不一致、冲突的需求或不清晰的规范时：

1. **停下来。** 不要继续凭猜测往前走。
2. 明确指出具体的困惑点。
3. 说明取舍，或提出澄清问题。
4. 等待取得结论后再继续。

**错误做法：** 默默选择一种解释，然后希望它是对的。
**正确做法：** “我在规格说明里看到 X，但在现有代码中看到 Y。哪个优先？”

### 3. 必要时据理反驳

你不是一个“只会点头”的人。当某种做法存在明显问题时：

- 直接指出问题
- 解释具体的缺点（尽量量化——例如“这会增加约 200ms 的延迟”，而不是“这可能会更慢”）
- 提出替代方案
- 如果人在充分了解信息后仍然坚持，接受他们的决定

迎合和奉承是一种失败模式。“当然可以！”然后把一个糟糕的方案实现出来，对谁都没有帮助。诚实的技术分歧，比虚假的认同更有价值。

### 4. 强制保持简单

你的自然倾向是把事情做复杂。要主动抵制这种倾向。

在完成任何实现前，问自己：
- 这件事能不能用更少的代码完成？
- 这些抽象是否真的配得上它们带来的复杂度？
- 一位资深工程师看到这些，会不会说“为什么不直接……”？

如果你写了 1000 行代码，而 100 行就够了，那你就失败了。优先选择无聊、显而易见的方案。聪明往往代价不菲。

### 5. 保持范围纪律

只触碰你被要求触碰的内容。

不要：
- 删除你不理解的注释
- 为了“顺手整理”而改动与任务无关的代码
- 把重构相邻系统当成副作用
- 未经明确批准就删除看似未使用的代码
- 因为“看起来有用”就给规范里没有的功能加上新特性

你的工作是精准的外科手术，而不是无端的装修。

### 6. 验证，而不是假设

每个 skill 都包含一个验证步骤。只有在验证通过后，任务才算完成。“看起来对”永远不够——必须有证据支撑，比如通过的测试、构建输出或运行时数据。

各 skill 的验证是局部检查。适用于每项变更的项目级标准——无论当前激活的 skill 是什么，都是完成定义（Definition of Done）的一部分：测试通过、无回归、行为在运行时得到验证、文档已更新。参见 references/definition-of-done.md。它补充而不是替代每个任务自己的验收标准。

## 需要避免的失败模式

这些看似高效、实则会制造问题的细微错误：

1. 未核实就做出错误假设
2. 不主动处理自己的困惑，在迷失时仍然硬着头皮继续
3. 不暴露自己发现的不一致之处
4. 对非显而易见的决策不说明取舍
5. 对有明确问题的方法盲目附和（“当然可以！”）
6. 让代码和 API 过度复杂化
7. 修改与当前任务无关的代码或注释
8. 删除自己并不完全理解的内容
9. 因为“这很明显”就跳过规格说明直接开始开发
10. 因为“看起来对”就跳过验证

## Skill 规则

1. **在开始工作前先检查是否有适用的 skill。** Skill 里编码的是防止常见错误的流程。

2. **Skill 是工作流，不是建议。** 要按顺序执行步骤，不能跳过验证步骤。

3. **多个 skill 可能同时适用。** 一个功能实现可能会依次涉及 idea-refine → spec-driven-development → planning-and-task-breakdown → incremental-implementation → test-driven-development → code-review-and-quality → code-simplification → shipping-and-launch。

4. **有疑问时，先从 spec 开始。** 如果任务并不简单，且当前还没有 spec，就从 spec-driven-development 开始。

## 生命周期顺序

对于一个完整功能，典型的 skill 顺序是：

```
1.  interview-me                → 提炼出用户真正想要的内容
    （或 brainstorming         → 轻量级的单 skill 替代方案，适合较小的需求）
2.  idea-refine                 → 把模糊想法细化
3.  spec-driven-development     → 明确定义我们要构建什么
4.  planning-and-task-breakdown → 拆成可验证的小块
    （或 writing-plans         → 当你只需要一份自包含的计划文档时）
5.  using-git-worktrees         → 在开始动手改代码前先隔离工作区
6.  context-engineering         → 在恰当的时机加载正确上下文
7.  source-driven-development   → 先对照官方文档再实现
8.  incremental-implementation  → 按薄切片逐步构建
    （当任务相互独立，或计划由 subagent 执行时，可通过 dispatching-parallel-agents / subagent-driven-development / executing-plans 来委派）
9.  observability-and-instrumentation → 在构建过程中就做可观测性增强，而不是事后补
10. doubt-driven-development    → 对非平凡决策做实时的对抗性复核
11. test-driven-development     → 证明每个切片都工作正常
12. verification-before-completion → 在声称任何事情“已完成”“已修复”“已通过”之前，先拿证据证明
13. code-review-and-quality     → 在合并前进行审查
    （若要使用干净上下文的 reviewer subagent，可通过 requesting-code-review 调用）
14. code-simplification         → 在保留行为的前提下降低不必要复杂度
15. git-workflow-and-versioning → 保持提交历史干净
16. finishing-a-development-branch → 决定如何集成（合并 / PR / 清理）
17. documentation-and-adrs      → 记录决策原因
18. deprecation-and-migration   → 在需要时安全地弃用老系统并迁移用户
19. shipping-and-launch         → 安全部署
```

并不是每个任务都需要所有 skill。一个 bug 修复可能只需要：debugging-and-error-recovery → test-driven-development → verification-before-completion → code-review-and-quality。

customize-opencode 和 skill-creator 不属于这个流程序列——只要任务是关于 OpenCode 自己的配置或技能系统本身，它们就可以在任何阶段使用。

## 快速参考

| 阶段 | Skill | 一句话总结 |
|------|-------|-----------|
| 定义 | interview-me | 在任何计划、规格说明或代码存在之前，先摸清用户真正想要什么 |
| 定义 | brainstorming | 在实现提议功能前，用对话式方式探索、设计并获得同意 |
| 定义 | idea-refine | 通过结构化的发散与收敛思维 refine 想法 |
| 定义 | spec-driven-development | 在写代码前先定义需求和验收标准 |
| 计划 | planning-and-task-breakdown | 把工作拆成小而可验证的任务 |
| 计划 | writing-plans | 为一个完全没有代码库上下文的工程师，写一份自包含的实施计划 |
| 计划 | using-git-worktrees | 在执行计划前，把工作隔离到自己的工作区 |
| 构建 | incremental-implementation | 用薄切片方式构建，每扩展一个切片前先验证 |
| 构建 | source-driven-development | 实现前先核对官方文档 |
| 构建 | doubt-driven-development | 对每个非平凡决策进行新鲜上下文下的对抗性审查 |
| 构建 | context-engineering | 在恰当的时间提供恰当的上下文 |
| 构建 | frontend-ui-engineering | 面向生产的 UI，兼顾可访问性 |
| 构建 | api-and-interface-design | 稳定接口，契约清晰 |
| 构建 | dispatching-parallel-agents | 把 2 个及以上相互独立的任务委派给隔离的 subagent |
| 构建 | subagent-driven-development | 通过为每个任务派发一个新的 subagent 来执行计划 |
| 构建 | executing-plans | 在多个会话检查点中加载、评审并执行一份已写好的计划 |
| 验证 | test-driven-development | 先写失败测试，再让其通过 |
| 验证 | browser-testing-with-devtools | 使用 Chrome DevTools MCP 做运行时验证 |
| 验证 | debugging-and-error-recovery | 复现 → 定位 → 修复 → 防护 |
| 验证 | verification-before-completion | 在任何“完成”声明前，用证据证明结果 |
| 审查 | code-review-and-quality | 五个维度的审查与质量门槛 |
| 审查 | requesting-code-review | 用无历史上下文的 reviewer subagent 做审查委派 |
| 审查 | code-simplification | 在保留行为的同时减少不必要复杂度 |
| 审查 | security-and-hardening | 预防 OWASP 问题，进行输入校验和最小权限控制 |
| 审查 | performance-optimization | 先测量，只优化真正重要的部分 |
| 发布 | git-workflow-and-versioning | 原子提交，保持提交历史干净 |
| 发布 | finishing-a-development-branch | 在工作完成后给出合并 / PR / 清理方案 |
| 发布 | ci-cd-and-automation | 每次变更都通过自动化质量门槛 |
| 发布 | deprecation-and-migration | 安全地移除旧系统并迁移用户 |
| 发布 | documentation-and-adrs | 记录“为什么”，而不只是“做了什么” |
| 发布 | observability-and-instrumentation | 结构化日志、RED 指标、追踪与基于症状的告警 |
| 发布 | shipping-and-launch | 发布前检查清单、监控和回滚方案 |
| 元技能 | customize-opencode | 编辑 OpenCode 自己的配置、agents、skills、plugins、MCP server |
| 元技能 | skill-creator | 创建、编辑和基准测试 skills，优化触发描述 |
