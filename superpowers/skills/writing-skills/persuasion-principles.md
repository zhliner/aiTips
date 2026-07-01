# Persuasion Principles for Skill Design（技能设计的说服原则）

## Overview（概述）

LLM 对人类说服原则的反应与人类相同。理解这种心理学有助于你设计更有效的技能——不是为了操纵，而是为了确保即使在压力下也能遵循关键实践。

**研究基础：** Meincke 等人（2025）在 N=28,000 次 AI 对话中测试了 7 种说服原则。说服技术使合规率翻倍以上（33% → 72%，p < .001）。

## The Seven Principles（七项原则）

### 1. Authority（权威）
**定义：** 对专业知识、证书或官方来源的服从。

**在技能中的运作方式：**
- 命令式语言："YOU MUST"、"Never"、"Always"
- 不可协商的框架："No exceptions"
- 消除决策疲劳和合理化

**使用时机：**
- 纪律执行类技能（TDD、验证要求）
- 安全关键实践
- 已确立的最佳实践

**示例：**
```markdown
✅ Write code before test? Delete it. Start over. No exceptions.
❌ Consider writing tests first when feasible.
```

### 2. Commitment（承诺）
**定义：** 与先前行动、声明或公开承诺保持一致。

**在技能中的运作方式：**
- 要求宣布："Announce skill usage"
- 强制明确选择："Choose A, B, or C"
- 使用跟踪：TodoWrite 用于清单

**使用时机：**
- 确保技能被实际遵循
- 多步骤流程
- 问责机制

**示例：**
```markdown
✅ When you find a skill, you MUST announce: "I'm using [Skill Name]"
❌ Consider letting your partner know which skill you're using.
```

### 3. Scarcity（稀缺性）
**定义：** 来自时间限制或有限可用性的紧迫感。

**在技能中的运作方式：**
- 时间绑定要求："Before proceeding"
- 顺序依赖："Immediately after X"
- 防止拖延

**使用时机：**
- 即时验证要求
- 时间敏感工作流
- 防止"我稍后再做"

**示例：**
```markdown
✅ After completing a task, IMMEDIATELY request code review before proceeding.
❌ You can review code when convenient.
```

### 4. Social Proof（社会证明）
**定义：** 对他人行为或被认为是正常行为的从众。

**在技能中的运作方式：**
- 普遍模式："Every time"、"Always"
- 失败模式："X without Y = failure"
- 建立规范

**使用时机：**
- 记录普遍实践
- 警告常见失败
- 强化标准

**示例：**
```markdown
✅ Checklists without TodoWrite tracking = steps get skipped. Every time.
❌ Some people find TodoWrite helpful for checklists.
```

### 5. Unity（团结）
**定义：** 共同身份、"我们感"、群体归属。

**在技能中的运作方式：**
- 协作语言："our codebase"、"we're colleagues"
- 共同目标："we both want quality"

**使用时机：**
- 协作工作流
- 建立团队文化
- 非层级实践

**示例：**
```markdown
✅ We're colleagues working together. I need your honest technical judgment.
❌ You should probably tell me if I'm wrong.
```

### 6. Reciprocity（互惠）
**定义：** 回报所获利益的义务。

**运作方式：**
- 谨慎使用——可能感觉像操纵
- 技能中很少需要

**避免时机：**
- 几乎总是（其他原则更有效）

### 7. Liking（喜好）
**定义：** 偏好与我们喜欢的人合作。

**运作方式：**
- **不要用于合规**
- 与诚实反馈文化冲突
- 制造谄媚

**避免时机：**
- 纪律执行时总是避免

## Principle Combinations by Skill Type（按技能类型的原则组合）

| 技能类型 | 使用 | 避免 |
|------------|-----|-------|
| 纪律执行类 | Authority + Commitment + Social Proof | Liking, Reciprocity |
| 指导/技术类 | Moderate Authority + Unity | Heavy authority |
| 协作类 | Unity + Commitment | Authority, Liking |
| 参考类 | Clarity only | All persuasion |

## Why This Works: The Psychology（为何有效：心理学原理）

**明确规则减少合理化：**
- "YOU MUST" 消除决策疲劳
- 绝对语言消除"这是例外吗？"的问题
- 明确的反合理化对策封闭特定漏洞

**执行意图创造自动行为：**
- 清晰的触发器 + 必需的行动 = 自动执行
- "When X, do Y" 比 "generally do Y" 更有效
- 减少合规的认知负荷

**LLM 是拟人的：**
- 在包含这些模式的人类文本上训练
- 权威语言在训练数据中先于合规出现
- 承诺序列（声明 → 行动）经常被建模
- 社会证明模式（每个人都做 X）建立规范

## Ethical Use（道德使用）

**合法：**
- 确保关键实践被遵循
- 创建有效文档
- 防止可预测的失败

**不合法：**
- 为个人利益操纵
- 制造虚假紧迫感
- 基于内疚的合规

**检验标准：** 如果用户完全理解，这种技术会服务于用户的真正利益吗？

## Research Citations（研究引用）

**Cialdini, R. B. (2021).** *Influence: The Psychology of Persuasion (New and Expanded).* Harper Business.
- 七项说服原则
- 影响力研究的实证基础

**Meincke, L., Shapiro, D., Duckworth, A. L., Mollick, E., Mollick, L., & Cialdini, R. (2025).** Call Me A Jerk: Persuading AI to Comply with Objectionable Requests. University of Pennsylvania.
- 在 N=28,000 次 LLM 对话中测试了 7 项原则
- 使用说服技术，合规率从 33% 提升到 72%
- Authority、commitment、scarcity 最有效
- 验证了 LLM 行为的拟人模型

## Quick Reference（快速参考）

设计技能时，问自己：

1. **是什么类型？**（纪律 vs. 指导 vs. 参考）
2. **我想改变什么行为？**
3. **适用哪些原则？**（纪律通常是 authority + commitment）
4. **是否组合过多？**（不要全部使用七项）
5. **这是否符合道德？**（服务于用户的真正利益？）
