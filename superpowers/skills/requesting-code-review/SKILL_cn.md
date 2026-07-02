---
name: requesting-code-review
description: 在完成任务、实现主要功能或合并前使用，以验证工作成果是否符合需求
---

# 请求代码审查

派遣代码审查子代理，在问题扩散之前捕获它们。审阅者获得精确构建的评估上下文——永远不会是你会话的历史记录。这使审阅者专注于工作成果而非你的思考过程，同时为你自己的继续工作保留上下文空间。

**核心原则：** 及早审查，经常审查。

## 何时请求审查

**强制性：**
- 在子代理驱动开发中的每个任务之后
- 完成主要功能之后
- 合并到 main 之前

**可选但有价值：**
- 卡住时（新的视角）
- 重构之前（基线检查）
- 修复复杂 bug 之后

## 如何请求

**1. 获取 git SHA：**
```bash
BASE_SHA=$(git rev-parse HEAD~1)  # 或 origin/main
HEAD_SHA=$(git rev-parse HEAD)
```

**2. 派遣代码审查子代理：**

派遣一个 `general-purpose` 子代理，填写 [code-reviewer.md](code-reviewer.md) 中的模板

**占位符：**
- `{DESCRIPTION}`——你构建内容的简要摘要
- `{PLAN_OR_REQUIREMENTS}`——它应该做什么
- `{BASE_SHA}`——起始提交
- `{HEAD_SHA}`——结束提交

**3. 根据反馈采取行动：**
- 立即修复 Critical（严重）问题
- 继续之前修复 Important（重要）问题
- 记录 Minor（次要）问题供稍后处理
- 如果审阅者有误，进行反驳（附上理由）

## 示例

```
[刚刚完成任务 2：添加验证函数]

你：让我在进行下一步之前请求代码审查。

BASE_SHA=$(git log --oneline | grep "任务 1" | head -1 | awk '{print $1}')
HEAD_SHA=$(git rev-parse HEAD)

[派遣代码审查子代理]
  DESCRIPTION：添加了 verifyIndex() 和 repairIndex()，包含 4 种问题类型
  PLAN_OR_REQUIREMENTS：来自 docs/superpowers/plans/deployment-plan.md 的任务 2
  BASE_SHA：a7981ec
  HEAD_SHA：3df7661

[子代理返回]：
  优点：架构清晰，真实的测试
  问题：
    Important：缺少进度指示器
    Minor：报告间隔使用了魔数（100）
  评估：可以继续

你：[修复进度指示器]
[继续任务 3]
```

## 与工作流的集成

**子代理驱动开发：**
- 每个任务之后进行审查
- 在问题叠加之前捕获它们
- 修复后再进入下一个任务

**执行计划：**
- 每个任务之后或在自然检查点进行审查
- 获取反馈，应用，继续

**临时开发：**
- 合并前审查
- 卡住时审查

## 红线

**绝不要：**
- 因为"很简单"而跳过审查
- 忽略 Critical（严重）问题
- 在未修复 Important（重要）问题的情况下继续
- 与有效的技术反馈争论

**如果审阅者有误：**
- 用技术推理反驳
- 展示证明其有效的代码/测试
- 请求澄清

请参阅模板：[code-reviewer.md](code-reviewer.md)
