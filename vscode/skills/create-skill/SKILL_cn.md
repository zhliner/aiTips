---
name: create-skill
description: '创建一个可复用的 skill（SKILL.md），将工作流打包封装。'
argument-hint: 此 skill 应产出什么？
disable-model-invocation: true
---
相关 skill：`agent-customization`。加载并遵循 **skills.md** 中的模板和原则。

引导用户创建一个 `SKILL.md`。

## 从对话中提取
首先，回顾对话历史。如果用户一直在遵循多步骤工作流或方法论（例如，调试方法、审查清单、实现模式），将其泛化为一个可复用的 skill。提取：
- 所遵循的分步流程
- 决策点和分支逻辑
- 质量标准或完成检查

## 必要时澄清
如果对话中没有明确出现工作流，进行澄清：
- 此 skill 应产出什么结果？
- workspace 级别还是个人级别？
- 简单的检查清单还是完整的多步骤工作流？

## 迭代
1. 起草 skill 并保存。
2. 找出最模糊或最薄弱的部分，并就此提问。
3. 确定后，总结 skill 的产出，建议示例提示词来试用它，并提出接下来要创建的相关自定义项。

记住遵循 `agent-customization` 指南来创建高效的 skill。
