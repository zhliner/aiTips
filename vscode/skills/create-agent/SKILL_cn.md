---
name: create-agent
description: '为特定任务创建一个自定义 agent（.agent.md）。'
argument-hint: 此 agent 应执行什么任务，如何执行？
disable-model-invocation: true
---
相关 skill：`agent-customization`。加载并遵循 **agents.md** 中的模板和原则。

引导用户创建一个 `.agent.md`。

## 从对话中提取
首先，回顾对话历史。如果用户一直以专业化的方式使用 agent（例如，限制工具、遵循特定角色、专注于某些文件类型），将其泛化为一个自定义 agent。提取：
- 所扮演的专业化角色或人设
- 工具偏好（使用哪些，避免哪些）
- 领域或任务范围

## 必要时澄清
如果对话中没有明确出现专业化方向，进行澄清：
- 此 agent 应执行什么任务？
- 何时应优先选择它而非默认 agent？
- 它应使用（或避免）哪些工具？

## 迭代
1. 起草 agent 文件并保存。
2. 找出最模糊或最薄弱的部分，并就此提问。
3. 确定后，总结 agent 的功能，建议示例提示词来试用它，并提出接下来要创建的相关自定义项。

记住遵循 `agent-customization` 指南来创建高效的 agent。
