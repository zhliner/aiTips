---
name: receiving-code-review
description: Use when receiving code review feedback, before implementing suggestions, especially if feedback seems unclear or technically questionable - requires technical rigor and verification, not performative agreement or blind implementation
---

# Code Review Reception（代码评审接收）

## 概述

代码评审需要技术评估，而非情绪表演。

**核心原则：** 先验证再实现。先提问再假设。技术正确性优先于社交舒适。

## 响应模式

```
WHEN receiving code review feedback:

1. READ: Complete feedback without reacting
2. UNDERSTAND: Restate requirement in own words (or ask)
3. VERIFY: Check against codebase reality
4. EVALUATE: Technically sound for THIS codebase?
5. RESPOND: Technical acknowledgment or reasoned pushback
6. IMPLEMENT: One item at a time, test each
```

## 禁止的回应

**绝对不要：**
- "You're absolutely right!"（明确违反 CLAUDE.md）
- "Great point!" / "Excellent feedback!"（表演性的）
- "Let me implement that now"（在验证之前）

**应该这样做：**
- 重述技术需求
- 提出澄清问题
- 如果有误，用技术理由反驳
- 直接开始工作（行动 > 言语）

## 处理不清晰的反馈

```
IF any item is unclear:
  STOP - do not implement anything yet
  ASK for clarification on unclear items

WHY: Items may be related. Partial understanding = wrong implementation.
```

**示例：**
```
your human partner: "Fix 1-6"
You understand 1,2,3,6. Unclear on 4,5.

❌ WRONG: Implement 1,2,3,6 now, ask about 4,5 later
✅ RIGHT: "I understand items 1,2,3,6. Need clarification on 4 and 5 before proceeding."
```

## 按来源分类处理

### 来自你的人类搭档
- **可信** - 理解后执行
- 如果范围不清晰**仍需提问**
- **不要表演性地同意**
- **直接行动**或技术性确认

### 来自外部评审者
```
BEFORE implementing:
  1. Check: Technically correct for THIS codebase?
  2. Check: Breaks existing functionality?
  3. Check: Reason for current implementation?
  4. Check: Works on all platforms/versions?
  5. Check: Does reviewer understand full context?

IF suggestion seems wrong:
  Push back with technical reasoning

IF can't easily verify:
  Say so: "I can't verify this without [X]. Should I [investigate/ask/proceed]?"

IF conflicts with your human partner's prior decisions:
  Stop and discuss with your human partner first
```

**你的人类搭档的规则：** "External feedback - be skeptical, but check carefully"

## 对"专业"功能的 YAGNI 检查

```
IF reviewer suggests "implementing properly":
  grep codebase for actual usage

  IF unused: "This endpoint isn't called. Remove it (YAGNI)?"
  IF used: Then implement properly
```

**你的人类搭档的规则：** "You and reviewer both report to me. If we don't need this feature, don't add it."

## 实现顺序

```
FOR multi-item feedback:
  1. Clarify anything unclear FIRST
  2. Then implement in this order:
     - Blocking issues (breaks, security)
     - Simple fixes (typos, imports)
     - Complex fixes (refactoring, logic)
  3. Test each fix individually
  4. Verify no regressions
```

## 何时反驳

在以下情况反驳：
- 建议会破坏现有功能
- 评审者缺乏完整上下文
- 违反 YAGNI（未使用的功能）
- 对此技术栈来说技术上不正确
- 存在遗留/兼容性原因
- 与你的人类搭档的架构决策冲突

**如何反驳：**
- 使用技术理由，而非防御性态度
- 提出具体问题
- 引用有效的测试/代码
- 如果涉及架构问题，让你的人类搭档参与

**如果你不便于公开反驳的信号：** "Strange things are afoot at the Circle K"

## 确认正确的反馈

当反馈确实正确时：
```
✅ "Fixed. [Brief description of what changed]"
✅ "Good catch - [specific issue]. Fixed in [location]."
✅ [直接修复并在代码中展示]

❌ "You're absolutely right!"
❌ "Great point!"
❌ "Thanks for catching that!"
❌ "Thanks for [anything]"
❌ 任何感谢的表达
```

**为什么不说感谢：** 行动说明一切。直接修复。代码本身就表明你听到了反馈。

**如果你发现自己在写"Thanks"：** 删掉它。改为陈述修复内容。

## 优雅地纠正你的反驳

如果你反驳了但你是错的：
```
✅ "You were right - I checked [X] and it does [Y]. Implementing now."
✅ "Verified this and you're correct. My initial understanding was wrong because [reason]. Fixing."

❌ 长篇道歉
❌ 为自己反驳辩护
❌ 过度解释
```

陈述纠正事实然后继续。

## 常见错误

| 错误 | 修正 |
|------|------|
| 表演性同意 | 陈述需求或直接行动 |
| 盲目实现 | 先对照代码库验证 |
| 批量处理不测试 | 逐个处理，逐个测试 |
| 假设评审者是对的 | 检查是否会破坏东西 |
| 回避反驳 | 技术正确性 > 舒适感 |
| 部分实现 | 先澄清所有项目 |
| 无法验证但仍继续 | 说明限制，请求指导 |

## 真实示例

**表演性同意（不好）：**
```
Reviewer: "Remove legacy code"
❌ "You're absolutely right! Let me remove that..."
```

**技术验证（好）：**
```
Reviewer: "Remove legacy code"
✅ "Checking... build target is 10.15+, this API needs 13+. Need legacy for backward compat. Current impl has wrong bundle ID - fix it or drop pre-13 support?"
```

**YAGNI（好）：**
```
Reviewer: "Implement proper metrics tracking with database, date filters, CSV export"
✅ "Grepped codebase - nothing calls this endpoint. Remove it (YAGNI)? Or is there usage I'm missing?"
```

**不清晰的项目（好）：**
```
your human partner: "Fix items 1-6"
You understand 1,2,3,6. Unclear on 4,5.
✅ "Understand 1,2,3,6. Need clarification on 4 and 5 before implementing."
```

## GitHub 线程回复

在 GitHub 上回复行内评审评论时，在评论线程中回复（`gh api repos/{owner}/{repo}/pulls/{pr}/comments/{id}/replies`），而非作为顶级 PR 评论。

## 底线

**外部反馈 = 需要评估的建议，而非需要执行的命令。**

验证。质疑。然后实现。

不要表演性同意。始终保持技术严谨。