---
name: requesting-code-review
description: 在完成任务、实现主要功能或合并前使用，用于验证工作是否满足要求
---

# 请求 Code Review

派遣一个 code reviewer subagent 来在问题扩散之前捕获它们。reviewer 获得精心构造的上下文用于评估——而非你的 session 历史。这使 reviewer 专注于工作成果，而非你的思考过程，同时保留你自己的上下文以便继续工作。

**核心原则：** 尽早 review，频繁 review。

## 何时请求 Review

**必须：**
- 在 subagent-driven development 中每个任务完成后
- 完成主要功能后
- 合并到 main 之前

**可选但有价值：**
- 卡住时（获取新视角）
- 重构前（基线检查）
- 修复复杂 bug 后

## 如何请求

**1. 获取 git SHA：**
```bash
BASE_SHA=$(git rev-parse HEAD~1)  # 或 origin/main
HEAD_SHA=$(git rev-parse HEAD)
```

**2. 派遣 code reviewer subagent：**

派遣一个 `general-purpose` subagent，填写 [code-reviewer.md](code-reviewer.md) 中的模板

**占位符：**
- `{DESCRIPTION}` - 你所构建内容的简要概述
- `{PLAN_OR_REQUIREMENTS}` - 它应该做什么
- `{BASE_SHA}` - 起始 commit
- `{HEAD_SHA}` - 结束 commit

**3. 根据反馈行动：**
- 立即修复 Critical 问题
- 在继续之前修复 Important 问题
- 记录 Minor 问题留待后续处理
- 如果 reviewer 有误，提出反驳（附理由）

## 示例

```
[刚完成任务 2：添加验证函数]

You: 让我在继续之前请求 code review。

BASE_SHA=$(git log --oneline | grep "Task 1" | head -1 | awk '{print $1}')
HEAD_SHA=$(git rev-parse HEAD)

[派遣 code reviewer subagent]
  DESCRIPTION: 添加了 verifyIndex() 和 repairIndex()，支持 4 种问题类型
  PLAN_OR_REQUIREMENTS: 来自 docs/superpowers/plans/deployment-plan.md 的任务 2
  BASE_SHA: a7981ec
  HEAD_SHA: 3df7661

[Subagent 返回]：
  Strengths: 架构清晰，测试真实
  Issues:
    Important: 缺少进度指示器
    Minor: 报告间隔使用了魔术数字 (100)
  Assessment: 可以继续

You: [修复进度指示器]
[继续任务 3]
```

## 与工作流的集成

**Subagent-Driven Development：**
- 每个任务完成后 review
- 在问题累积之前捕获
- 修复后再进入下一个任务

**Executing Plans：**
- 每个任务后或自然检查点 review
- 获取反馈，应用，继续

**Ad-Hoc Development：**
- 合并前 review
- 卡住时 review

## 危险信号

**绝不：**
- 因为"很简单"而跳过 review
- 忽略 Critical 问题
- 带着未修复的 Important 问题继续
- 对有效的技术反馈争辩

**如果 reviewer 有误：**
- 用技术理由反驳
- 展示证明其有效的代码/测试
- 请求澄清

参见模板：[code-reviewer.md](code-reviewer.md)
