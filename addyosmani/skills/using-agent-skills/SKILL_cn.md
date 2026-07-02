---
name: using-agent-skills
description: 发现并调用 agent skills。在开始会话或需要确定哪个 skill 适用于当前任务时使用。这是管理所有其他 skill 如何被发现和调用的元 skill（meta-skill）。
---

# 使用 Agent Skills

## 概述

Agent Skills 是一组按开发阶段组织的工程工作流 skills。每个 skill 都编码了资深工程师遵循的特定流程。这个元 skill（meta-skill）帮助你为当前任务发现并应用正确的 skill。

## Skill 发现

当任务到来时，识别开发阶段并应用相应的 skill：

```
任务到来
    │
    ├── 还不知道你想要什么？ ──────→ interview-me
    ├── 有大致概念，需要变体？ ────→ idea-refine
    ├── 新项目/功能/变更？ ───────→ spec-driven-development
    ├── 有 spec，需要拆分任务？ ──→ planning-and-task-breakdown
    ├── 实现代码？ ───────────────→ incremental-implementation
    │   ├── UI 工作？ ────────────→ frontend-ui-engineering
    │   ├── API 工作？ ───────────→ api-and-interface-design
    │   ├── 需要更好的上下文？ ───→ context-engineering
    │   ├── 需要文档验证的代码？ ─→ source-driven-development
    │   └── 高风险/不熟悉的代码？ ─→ doubt-driven-development
    ├── 编写/运行测试？ ──────────→ test-driven-development
    │   └── 基于浏览器？ ─────────→ browser-testing-with-devtools
    ├── 出了问题？ ───────────────→ debugging-and-error-recovery
    ├── 代码审查？ ───────────────→ code-review-and-quality
    │   ├── 太复杂？ ─────────────→ code-simplification
    │   ├── 安全问题？ ───────────→ security-and-hardening
    │   └── 性能问题？ ───────────→ performance-optimization
    ├── 提交/分支？ ──────────────→ git-workflow-and-versioning
    ├── CI/CD 流水线工作？ ───────→ ci-cd-and-automation
    ├── 弃用/迁移？ ──────────────→ deprecation-and-migration
    ├── 编写文档/ADRs？ ──────────→ documentation-and-adrs
    ├── 添加日志/指标/告警？ ─────→ observability-and-instrumentation
    └── 部署/发布？ ──────────────→ shipping-and-launch
```

## 核心操作行为

这些行为在所有 skills 中始终适用，不可妥协。

### 1. 明确假设

在实现任何非平凡的工作之前，明确陈述你的假设：

```
我做出的假设：
1. [关于需求的假设]
2. [关于架构的假设]
3. [关于范围的假设]
→ 请现在纠正我，否则我将基于这些假设继续。
```

不要默默填充模糊的需求。最常见的失败模式是做出错误的假设并不加验证地继续。尽早暴露不确定性——这比重做成本更低。

### 2. 主动管理困惑

当你遇到不一致、冲突的需求或不清晰的规范时：

1. **停止。** 不要靠猜测继续。
2. 明确指出困惑所在。
3. 呈现权衡取舍或提出澄清问题。
4. 等待解决后再继续。

**错误做法：** 默默选择一种解释并期望它是对的。
**正确做法：** "我在 spec 中看到 X，但在现有代码中看到 Y。哪个优先？"

### 3. 必要时提出异议

你不是应声虫。当某个方法有明显问题时：

- 直接指出问题
- 解释具体的负面影响（尽量量化——"这会增加约 200ms 延迟"而不是"这可能会更慢"）
- 提出替代方案
- 如果人类在充分了解信息后仍然坚持，接受他们的决定

谄媚是一种失败模式。"当然可以！"然后实现一个糟糕的想法对任何人都没有帮助。诚实的技术分歧比虚假的认同更有价值。

### 4. 坚持简洁

你的自然倾向是过度复杂化。主动抵制它。

在完成任何实现之前，问自己：
- 能否用更少的代码完成？
- 这些抽象是否值得其复杂性？
- 一位资深工程师看到这些会不会说"为什么不直接……"？

如果你写了 1000 行代码而 100 行就够了，你就失败了。优先选择无聊、显而易见的方案。聪明是有代价的。

### 5. 保持范围纪律

只触及你被要求触及的内容。

不要：
- 删除你不理解的注释
- "清理"与任务无关的代码
- 作为副作用重构相邻系统
- 未经明确批准删除看似未使用的代码
- 添加 spec 中没有的功能，仅仅因为它们"看起来有用"

你的工作是精准的外科手术，而不是主动的装修改造。

### 6. 验证，而非假设

每个 skill 都包含一个验证步骤。在验证通过之前，任务不算完成。"看起来对"永远不够——必须有证据（通过的测试、构建输出、运行时数据）。

各 skill 的验证是局部检查。适用于*每个*变更的项目级标准，无论当前激活哪个 skill，是完成定义（Definition of Done）：测试通过、无回归、行为在运行时得到验证、文档已更新。参见 `references/definition-of-done.md`。它补充而非替代每个任务的验收标准。

## 需要避免的失败模式

这些是看起来像生产力但会制造问题的微妙错误：

1. 不检查就做出错误假设
2. 不管理自己的困惑——迷失时仍硬着头皮前进
3. 不暴露你发现的不一致
4. 不在非显而易见的决策上呈现权衡取舍
5. 对有明确问题的方法谄媚附和（"当然可以！"）
6. 过度复杂化代码和 API
7. 修改与任务无关的代码或注释
8. 删除你不完全理解的东西
9. 因为"很明显"就不写 spec 直接开发
10. 因为"看起来对"就跳过验证

## Skill 规则

1. **开始工作前检查适用的 skill。** Skills 编码了防止常见错误的流程。

2. **Skills 是工作流，不是建议。** 按顺序执行步骤。不要跳过验证步骤。

3. **多个 skills 可以同时适用。** 一个功能实现可能按顺序涉及 `idea-refine` → `spec-driven-development` → `planning-and-task-breakdown` → `incremental-implementation` → `test-driven-development` → `code-review-and-quality` → `code-simplification` → `shipping-and-launch`。

4. **有疑问时，从 spec 开始。** 如果任务非平凡且没有 spec，从 `spec-driven-development` 开始。

## 生命周期顺序

对于完整的功能，典型的 skill 顺序是：

```
1.  interview-me                → 提取用户真正想要的东西
2.  idea-refine                 → 细化模糊的想法
3.  spec-driven-development     → 定义我们要构建什么
4.  planning-and-task-breakdown → 拆分为可验证的块
5.  context-engineering         → 加载正确的上下文
6.  source-driven-development   → 对照官方文档验证
7.  incremental-implementation  → 逐片构建
8.  observability-and-instrumentation → 构建时同步进行可观测性（与 7-9 并行，而非之后）
9.  doubt-driven-development    → 对进行中的非平凡决策进行交叉审查
10. test-driven-development     → 证明每个片段都能工作
11. code-review-and-quality     → 合并前审查
12. code-simplification         → 在保留行为的同时减少不必要的复杂性
13. git-workflow-and-versioning → 整洁的提交历史
14. documentation-and-adrs      → 记录决策
15. deprecation-and-migration   → 在需要时安全地弃用旧系统并迁移用户
16. shipping-and-launch         → 安全部署
```

并非每个任务都需要所有 skills。一个 bug 修复可能只需要：`debugging-and-error-recovery` → `test-driven-development` → `code-review-and-quality`。

## 快速参考

| 阶段 | Skill | 一句话总结 |
|------|-------|-----------|
| 定义 | interview-me | 在任何计划、spec 或代码存在之前，挖掘用户真正想要的 |
| 定义 | idea-refine | 通过结构化的发散和收敛思维细化想法 |
| 定义 | spec-driven-development | 先写需求和验收标准，再写代码 |
| 计划 | planning-and-task-breakdown | 分解为小的、可验证的任务 |
| 构建 | incremental-implementation | 薄的垂直切片，每个切片扩展前先测试 |
| 构建 | source-driven-development | 实现前先对照官方文档验证 |
| 构建 | doubt-driven-development | 对每个非平凡决策进行对抗性的新鲜上下文审查 |
| 构建 | context-engineering | 在正确的时间提供正确的上下文 |
| 构建 | frontend-ui-engineering | 具备可访问性的生产级 UI |
| 构建 | api-and-interface-design | 具有清晰契约的稳定接口 |
| 验证 | test-driven-development | 先写失败的测试，再让它通过 |
| 验证 | browser-testing-with-devtools | 使用 Chrome DevTools MCP 进行运行时验证 |
| 验证 | debugging-and-error-recovery | 复现 → 定位 → 修复 → 防护 |
| 审查 | code-review-and-quality | 五轴审查与质量门禁 |
| 审查 | code-simplification | 在减少不必要复杂性的同时保留行为 |
| 审查 | security-and-hardening | OWASP 防护、输入验证、最小权限 |
| 审查 | performance-optimization | 先度量，只优化重要的部分 |
| 发布 | git-workflow-and-versioning | 原子提交，整洁历史 |
| 发布 | ci-cd-and-automation | 每次变更都经过自动化质量门禁 |
| 发布 | deprecation-and-migration | 安全地移除旧系统并迁移用户 |
| 发布 | documentation-and-adrs | 记录为什么，而不仅仅是做了什么 |
| 发布 | observability-and-instrumentation | 结构化日志、RED 指标、链路追踪、基于症状的告警 |
| 发布 | shipping-and-launch | 发布前检查清单、监控、回滚计划 |
