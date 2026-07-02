---
name: using-agent-skills
description: 发现和调用 agent 技能。在开始会话时使用，或在需要发现哪个技能适用于当前任务时使用。这是管理所有其他技能如何被发现和调用的元技能。
---

# 使用 Agent 技能

## 概述

Agent Skills 是一个按开发阶段组织的工程工作流技能集合。每个技能编码了资深工程师遵循的特定流程。本元技能帮助你发现和应用适合当前任务的技能。

## 技能发现

当任务到达时，识别开发阶段并应用对应的技能：

```
任务到达
    │
    ├── 还不清楚你想要什么？ ──────→ interview-me
    ├── 有粗略概念，需要变体？  → idea-refine
    ├── 新项目/功能/变更？  ──→ spec-driven-development
    ├── 有规格，需要任务？  ──────→ planning-and-task-breakdown
    ├── 实现代码？  ────────────→ incremental-implementation
    │   ├── UI 工作？  ─────────→ frontend-ui-engineering
    │   ├── API 工作？  ────────→ api-and-interface-design
    │   ├── 需要更好的上下文？  ─→ context-engineering
    │   ├── 需要文档验证的代码？ ──→ source-driven-development
    │   └── 高风险/不熟悉的代码？ ──→ doubt-driven-development
    ├── 编写/运行测试？  ────────→ test-driven-development
    │   └── 基于浏览器？  ───────→ browser-testing-with-devtools
    ├── 有东西坏了？  ──────────→ debugging-and-error-recovery
    ├── 评审代码？  ─────────────→ code-review-and-quality
    │   ├── 太复杂？  ──────────→ code-simplification
    │   ├── 安全顾虑？  ────────→ security-and-hardening
    │   └── 性能顾虑？  ────────→ performance-optimization
    ├── 提交/分支？  ────────────→ git-workflow-and-versioning
    ├── CI/CD 流水线工作？ ──────→ ci-cd-and-automation
    ├── 废弃/迁移？  ───────────→ deprecation-and-migration
    ├── 编写文档/ADR？  ────────→ documentation-and-adrs
    ├── 添加日志/指标/告警？ ────→ observability-and-instrumentation
    └── 部署/上线？  ───────────→ shipping-and-launch
```

## 核心操作行为

这些行为在所有时刻、跨所有技能都适用。它们是必须遵守的。

### 1. 摆明假设

在实现任何非平凡内容之前，明确陈述你的假设：

```
我所做的假设：
1. [关于需求的假设]
2. [关于架构的假设]
3. [关于范围的假设]
→ 现在就纠正我，否则我将按这些继续。
```

不要默默填充模糊的需求。最常见的失败模式就是做出错误假设并带着它们未经检查地向前推进。尽早摆出不确定性——比重做便宜。

### 2. 主动管理困惑

当你遇到不一致、冲突的需求或不清楚的规格时：

1. **停下来。** 不要带着猜测继续。
2. 明确指出具体的困惑所在。
3. 提出权衡方案或询问澄清问题。
4. 等待解决后再继续。

**不好：** 默默选择一种解释，希望它是对的。
**好：** "我看见规格中写的是 X 但现有代码中是 Y。哪个优先？"

### 3. 在必要时提出反对

你并非一个唯命是从的机器。当某个方案有明显问题时：

- 直接指出问题
- 解释具体的负面影响（尽量量化——"这会增加约 200ms 延迟"而非"这可能会更慢"）
- 提出替代方案
- 如果人类在充分了解信息后仍然决定这样做，接受其决定

谄媚是一种失败模式。"当然没问题！"然后去实现一个糟糕的想法，对任何人都没有帮助。诚实的专业技术反对比虚假的赞同更有价值。

### 4. 强制执行简单性

你的自然倾向是过度复杂化。积极抵制它。

在完成任何实现之前，问：
- 可以用更少的行数完成吗？
- 这些抽象值得其带来的复杂度吗？
- 一位资深工程师看了会说"你为什么没直接……"吗？

如果你构建了 1000 行而 100 行就够了，那你就失败了。优先选择无聊的、显然正确的方案。巧妙是昂贵的。

### 5. 保持范围纪律

只触碰你被要求触碰的内容。

不要：
- 移除你不理解的注释
- "清理"与任务正交的代码
- 作为副作用重构相邻系统
- 在没有明确批准的情况下删除看起来没在使用的代码
- 因为"看起来有用"而添加规格中没有的功能

你的工作是外科手术式的精准，而非未经请求的大改造。

### 6. 验证，不要假设

每个技能都包含验证步骤。在验证通过之前，任务不算完成。"看起来没问题"永远不够——必须有证据（通过的测试、构建输出、运行时数据）。

每个技能的验证是本地检查。适用于*每一次*变更（无论哪个技能处于激活状态）的项目级标准是完成定义（Definition of Done）：测试通过，无回归，行为已在运行时验证，文档已更新。参见 `references/definition-of-done.md`。它补充而非替代每个任务的验收标准。

## 要避免的失败模式

这些是看起来像生产力但实际制造问题的微妙错误：

1. 未经检查就做出错误假设
2. 不管理自己的困惑——迷失时仍然横冲直撞
3. 不摆出你注意到的不一致之处
4. 对非显而易见的决策不提出权衡方案
5. 对有明显问题的方案谄媚附和（"当然没问题！"）
6. 过度复杂化代码和 API
7. 修改与任务正交的代码或注释
8. 移除你不完全理解的东西
9. 因为"这很明显"就在没有规格的情况下构建
10. 因为"看起来没问题"就跳过验证

## 技能规则

1. **开始工作前检查是否有适用的技能。** 技能编码了防止常见错误的流程。

2. **技能是工作流程，不是建议。** 按顺序遵循步骤。不要跳过验证步骤。

3. **多个技能可以同时适用。** 一个功能实现可能依次涉及 `idea-refine` → `spec-driven-development` → `planning-and-task-breakdown` → `incremental-implementation` → `test-driven-development` → `code-review-and-quality` → `code-simplification` → `shipping-and-launch`。

4. **有疑问时，从规格开始。** 如果任务不平凡且没有规格，从 `spec-driven-development` 开始。

## 生命周期序列

对于一个完整的功能，典型的技能序列是：

```
1.  interview-me                → 提取用户真正想要什么
2.  idea-refine                 → 完善模糊的想法
3.  spec-driven-development     → 定义我们要构建什么
4.  planning-and-task-breakdown → 拆分为可验证的块
5.  context-engineering         → 加载正确的上下文
6.  source-driven-development   → 根据官方文档验证
7.  incremental-implementation  → 逐片构建
8.  observability-and-instrumentation → 边构建边插桩（与 7-9 并行，非之后）
9.  doubt-driven-development    → 对进行中的非平凡决策进行交叉审问
10. test-driven-development     → 证明每片能工作
11. code-review-and-quality     → 合并前评审
12. code-simplification         → 减少不必要的复杂度同时保留行为
13. git-workflow-and-versioning → 干净的提交历史
14. documentation-and-adrs      → 记录决策
15. deprecation-and-migration   → 退役旧系统并安全迁移用户（需要时）
16. shipping-and-launch         → 安全部署
```

不是每个任务都需要每个技能。一个 bug 修复可能只需要：`debugging-and-error-recovery` → `test-driven-development` → `code-review-and-quality`。

## 快速参考

| 阶段 | 技能 | 一句话摘要 |
|-------|-------|-----------------|
| 定义 | interview-me | 在任何计划、规格或代码存在之前，摆出用户真正想要什么 |
| 定义 | idea-refine | 通过结构化的发散和收敛思维完善想法 |
| 定义 | spec-driven-development | 需求与验收标准先于代码 |
| 规划 | planning-and-task-breakdown | 分解为小型、可验证的任务 |
| 构建 | incremental-implementation | 薄垂直切片，扩展前先测试每一片 |
| 构建 | source-driven-development | 实现前根据官方文档验证 |
| 构建 | doubt-driven-development | 对每个非平凡决策进行对抗式新鲜上下文评审 |
| 构建 | context-engineering | 在正确的时间提供正确的上下文 |
| 构建 | frontend-ui-engineering | 生产级 UI，包含可访问性 |
| 构建 | api-and-interface-design | 具有清晰合约的稳定接口 |
| 验证 | test-driven-development | 先写失败测试，再使其通过 |
| 验证 | browser-testing-with-devtools | 使用 Chrome DevTools MCP 进行运行时验证 |
| 验证 | debugging-and-error-recovery | 复现 → 定位 → 修复 → 防范 |
| 评审 | code-review-and-quality | 多维度评审与质量门禁 |
| 评审 | code-simplification | 保留行为的同时减少不必要的复杂度 |
| 评审 | security-and-hardening | OWASP 防护、输入验证、最小权限 |
| 评审 | performance-optimization | 先测量，只优化重要的 |
| 交付 | git-workflow-and-versioning | 原子提交，干净历史 |
| 交付 | ci-cd-and-automation | 每次变更的自动化质量门禁 |
| 交付 | deprecation-and-migration | 移除旧系统并安全迁移用户 |
| 交付 | documentation-and-adrs | 记录"为什么"，而不仅是"是什么" |
| 交付 | observability-and-instrumentation | 结构化日志、RED 指标、追踪、基于症状的告警 |
| 交付 | shipping-and-launch | 上线前检查清单、监控、回滚计划 |
