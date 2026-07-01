---
name: verification-before-completion
description: Use when about to claim work is complete, fixed, or passing, before committing or creating PRs - requires running verification commands and confirming output before making any success claims; evidence before assertions always
---

# Verification Before Completion（完成前验证）

## 概述

在没有验证的情况下声称工作已完成，是不诚实，而非高效。

**核心原则：** 证据先于断言，永远如此。

**违反本规则的字面含义，就是违反本规则的精神。**

## 铁律

```
NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE
```

如果你在这条消息中没有运行过验证命令，你就不能声称它通过了。

## 关卡函数

```
BEFORE claiming any status or expressing satisfaction:

1. IDENTIFY: What command proves this claim?
2. RUN: Execute the FULL command (fresh, complete)
3. READ: Full output, check exit code, count failures
4. VERIFY: Does output confirm the claim?
   - If NO: State actual status with evidence
   - If YES: State claim WITH evidence
5. ONLY THEN: Make the claim

Skip any step = lying, not verifying
```

## 常见失败

| 声称 | 需要 | 不充分的情况 |
|------|------|-------------|
| 测试通过 | 测试命令输出：0 失败 | 之前的运行、"应该能通过" |
| Linter 无错误 | Linter 输出：0 错误 | 部分检查、推断 |
| 构建成功 | 构建命令：exit 0 | Linter 通过、日志看起来正常 |
| Bug 已修复 | 测试原始症状：通过 | 代码已修改、假设已修复 |
| 回归测试有效 | 红-绿循环已验证 | 测试仅通过一次 |
| 代理已完成 | VCS diff 显示变更 | 代理报告"成功" |
| 需求已满足 | 逐条核对清单 | 测试通过 |

## 危险信号 - 立即停止

- 使用"应该"、"大概"、"似乎"
- 在验证之前表达满意（"太好了！"、"完美！"、"完成了！"等）
- 即将在没有验证的情况下提交/推送/创建 PR
- 信任代理的成功报告
- 依赖部分验证
- 想着"就这一次"
- 太累了，只想赶紧结束
- **任何暗示成功但未运行验证的措辞**

## 防止自我合理化

| 借口 | 现实 |
|------|------|
| "现在应该能用了" | 运行验证命令 |
| "我很有信心" | 信心 ≠ 证据 |
| "就这一次" | 没有例外 |
| "Linter 通过了" | Linter ≠ 编译器 |
| "代理说成功了" | 独立验证 |
| "我累了" | 疲惫 ≠ 借口 |
| "部分检查就够了" | 部分检查什么也证明不了 |
| "换了措辞所以规则不适用" | 精神高于字面 |

## 关键模式

**测试：**
```
✅ [运行测试命令] [看到：34/34 通过] "All tests pass"
❌ "Should pass now" / "Looks correct"
```

**回归测试（TDD 红-绿）：**
```
✅ 编写 → 运行（通过）→ 撤销修复 → 运行（必须失败）→ 恢复 → 运行（通过）
❌ "I've written a regression test"（未进行红-绿验证）
```

**构建：**
```
✅ [运行构建] [看到：exit 0] "Build passes"
❌ "Linter passed"（linter 不检查编译）
```

**需求：**
```
✅ 重新阅读计划 → 创建核对清单 → 逐项验证 → 报告差距或完成情况
❌ "Tests pass, phase complete"
```

**代理委托：**
```
✅ 代理报告成功 → 检查 VCS diff → 验证变更 → 报告实际状态
❌ 信任代理报告
```

## 为什么这很重要

来自 24 次失败记录：
- 你的人类搭档说"我不相信你" - 信任被打破
- 未定义的函数被交付 - 会导致崩溃
- 缺失的需求被交付 - 功能不完整
- 时间浪费在虚假的完成上 → 方向修正 → 返工
- 违反了："诚实是核心价值观。如果你撒谎，你会被替换。"

## 何时应用

**在以下操作之前始终执行：**
- 任何关于成功/完成的声称
- 任何表达满意的语句
- 任何关于工作状态的正向陈述
- 提交、创建 PR、标记任务完成
- 进入下一个任务
- 委托给代理

**规则适用于：**
- 精确的措辞
- 释义和同义词
- 对成功的暗示
- 任何暗示完成/正确的沟通

## 底线

**验证没有捷径。**

运行命令。阅读输出。然后才能声称结果。

这不可谈判。