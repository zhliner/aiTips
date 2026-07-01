---
name: requesting-code-review
description: Use when completing tasks, implementing major features, or before merging to verify work meets requirements
---

# Requesting Code Review（请求代码评审）

分派 superpowers:code-reviewer 子代理来在问题级联之前捕获它们。

**核心原则：** 早评审，常评审。

## 何时请求评审

**强制性：**
- 在子代理驱动开发中每个任务完成后
- 完成主要功能后
- 合并到 main 之前

**可选但有价值：**
- 卡住时（获取新视角）
- 重构前（基线检查）
- 修复复杂 bug 后

## 如何请求

**1. 获取 git SHA：**
```bash
BASE_SHA=$(git rev-parse HEAD~1)  # or origin/main
HEAD_SHA=$(git rev-parse HEAD)
```

**2. 分派 code-reviewer 子代理：**

使用 Task 工具，类型为 superpowers:code-reviewer，填写 `code-reviewer.md` 中的模板

**占位符：**
- `{WHAT_WAS_IMPLEMENTED}` - 你刚构建了什么
- `{PLAN_OR_REQUIREMENTS}` - 它应该做什么
- `{BASE_SHA}` - 起始提交
- `{HEAD_SHA}` - 结束提交
- `{DESCRIPTION}` - 简要总结

**3. 根据反馈行动：**
- 立即修复 Critical 级别问题
- 在继续之前修复 Important 级别问题
- 记录 Minor 级别问题留待后续处理
- 如果评审者有误，进行反驳（附带理由）

## 示例

```
[刚完成 Task 2: Add verification function]

You: Let me request code review before proceeding.

BASE_SHA=$(git log --oneline | grep "Task 1" | head -1 | awk '{print $1}')
HEAD_SHA=$(git rev-parse HEAD)

[分派 superpowers:code-reviewer 子代理]
  WHAT_WAS_IMPLEMENTED: Verification and repair functions for conversation index
  PLAN_OR_REQUIREMENTS: Task 2 from docs/plans/deployment-plan.md
  BASE_SHA: a7981ec
  HEAD_SHA: 3df7661
  DESCRIPTION: Added verifyIndex() and repairIndex() with 4 issue types

[子代理返回]:
  Strengths: Clean architecture, real tests
  Issues:
    Important: Missing progress indicators
    Minor: Magic number (100) for reporting interval
  Assessment: Ready to proceed

You: [Fix progress indicators]
[Continue to Task 3]
```

## 与工作流的集成

**子代理驱动开发：**
- 每个任务完成后评审
- 在问题复合之前捕获
- 修复后再进入下一个任务

**Executing Plans：**
- 每批（3 个任务）后评审
- 获取反馈，应用，继续

**临时开发：**
- 合并前评审
- 卡住时评审

## 危险信号

**绝对不要：**
- 以"很简单"为由跳过评审
- 忽略 Critical 级别问题
- 带着未修复的 Important 级别问题继续
- 与有效的技术反馈争论

**如果评审者有误：**
- 用技术理由反驳
- 展示证明其有效的代码/测试
- 请求澄清

参见模板：requesting-code-review/code-reviewer.md