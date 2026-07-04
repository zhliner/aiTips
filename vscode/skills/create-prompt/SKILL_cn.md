---
name: create-prompt
description: '为常见任务创建一个可复用的 prompt 文件（.prompt.md）。'
argument-hint: 此 prompt 应帮助完成什么任务？
disable-model-invocation: true
---
相关 skill：`agent-customization`。加载并遵循 **prompts.md** 中的模板和原则。

引导用户创建一个 `.prompt.md`。

## 从对话中提取
首先，回顾对话历史。如果用户一直在处理可重复的任务模式（例如，解释代码、生成测试、重构），将其泛化为一个可复用的 prompt。提取：
- 重复执行的核心任务
- 任何隐式输入（选中的代码、文件类型、上下文）
- 期望的输出格式或风格

## 必要时澄清
如果对话中没有明确出现模式，进行澄清：
- 此 prompt 应帮助完成什么任务？
- 它应接受参数还是使用固定上下文？
- workspace 级别还是个人级别？

## 迭代
1. 起草 prompt 并保存。
2. 找出最模糊或最薄弱的部分，并就此提问。
3. 确定后，总结 prompt 的功能，建议示例调用方式，并提出接下来要创建的相关自定义项。

记住遵循 `agent-customization` 指南来创建高效的 prompt。
