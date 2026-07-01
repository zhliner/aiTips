---
name: requesting-code-review
description: 在完成任务、实现主要功能或合并前使用，以验证工作是否符合需求
---

# 请求代码审查（Requesting Code Review）

调度一个代码审查 subagent，在问题连锁放大之前捕获它们。审查者会收到精心构建的评估上下文——而不是你的 session 历史。这使审查者聚焦于工作成果而非你的思考过程，同时保留你自己的上下文用于后续工作。

**核心原则：** 早审查，勤审查。

## 何时请求审查

**必须：**
- subagent-driven 开发中的每个任务完成后
- 完成主要功能后
- 合并到 main 之前

**可选但推荐：**
- 卡住时（新视角）
- 重构前（基准检查）
- 修复复杂 bug 后

## 如何请求

**1. 获取 git SHA：**
```bash
BASE_SHA=$(git rev-parse HEAD~1)  # 或 origin/main
HEAD_SHA=$(git rev-parse HEAD)
```

**2. 调度代码审查 subagent：**

调度一个 `general-purpose` subagent，填写 [code-reviewer.md](code-reviewer.md) 中的模板

**占位符：**
- `{DESCRIPTION}` — 你所构建内容的简要总结
- `{PLAN_OR_REQUIREMENTS}` — 它应该做什么
- `{BASE_SHA}` — 起始 commit
- `{HEAD_SHA}` — 结束 commit

**3. 根据反馈行动：**
- 立即修复 Critical 问题
- 在继续前修复 Important 问题
- 记下 Minor 问题稍后处理
- 如果审查者错了，合理反驳（附推理）

## 示例

```
[刚完成任务 2：添加验证函数]

你：让我在继续之前请求代码审查。

BASE_SHA=$(git log --oneline | grep "Task 1" | head -1 | awk '{print $1}')
HEAD_SHA=$(git rev-parse HEAD)

[调度代码审查 subagent]
  DESCRIPTION: 添加了支持 4 种问题类型的 verifyIndex() 和 repairIndex()
  PLAN_OR_REQUIREMENTS: docs/superpowers/plans/deployment-plan.md 中的任务 2
  BASE_SHA: a7981ec
  HEAD_SHA: 3df7661

[Subagent 返回]:
  优点: 架构清晰，测试真实
  问题:
    Important: 缺少进度指示器
    Minor: 报告间隔用了魔数 (100)
  评估: 可以继续

你：[修复进度指示器]
[继续任务 3]
```

## 与工作流的集成

**Subagent-Driven Development：**
- 每个任务后审查
- 在问题叠加之前捕获
- 修复后再进入下一个任务

**Executing Plans：**
- 每个任务后或在自然 checkpoint 处审查
- 获取反馈、应用、继续

**Ad-Hoc 开发：**
- 合并前审查
- 卡住时审查

## 红灯警告

**绝不：**
- 因为是"简单的"就跳过审查
- 忽略 Critical 问题
- 未修复 Important 问题就继续
- 与有效的技术反馈争辩

**如果审查者错了：**
- 以技术推理反驳
- 展示证明其正常工作的代码/测试
- 请求澄清

参见模板：[code-reviewer.md](code-reviewer.md)
