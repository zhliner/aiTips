# Code Quality Reviewer Prompt Template（代码质量审查员提示词模板）

在派遣代码质量审查子代理时使用此模板。

**目的：** 验证实现是否构建良好（整洁、经过测试、可维护）

**仅在规格合规审查通过后派遣。**

```
Task tool (superpowers:code-reviewer):
  Use template at requesting-code-review/code-reviewer.md

  WHAT_WAS_IMPLEMENTED: [来自实现者的报告]
  PLAN_OR_REQUIREMENTS: Task N from [plan-file]
  BASE_SHA: [任务前的提交]
  HEAD_SHA: [当前提交]
  DESCRIPTION: [任务摘要]
```

**代码审查员返回：** 优点、问题（严重/重要/轻微）、评估
