---
name: create-instructions
description: '为项目规则或约定创建一个 instructions 文件（.instructions.md）。'
argument-hint: 需要强制执行什么规则或约定？
disable-model-invocation: true
---
相关 skill：`agent-customization`。加载并遵循 **instructions.md** 中的模板和原则。

引导用户创建一个 instructions 文件。

## 从对话中提取
首先，回顾对话历史。如果用户一直在纠正 agent 的输出或要求特定模式（例如，"总是使用 X"、"从不做 Y"、"遵循这种风格"），将其泛化为一个持久化的 instruction。提取：
- 对话中提到的纠正或偏好
- 用户强制要求或请求的编码模式
- 引用的项目特定约定

## 必要时澄清
如果对话中没有明确出现规则，进行澄清：
- 这应适用于所有地方还是仅特定文件？
- 涉及哪些技术或文件类型？
- 这是一条硬性规则还是偏好？

如果需要更多上下文，使用 subagent 探索代码库。

## 迭代
1. 起草 instruction 并保存。
2. 找出最模糊或最薄弱的部分，并就此提问。
3. 确定后，总结 instruction 强制执行的内容，建议示例提示词来查看其效果，并提出接下来要创建的相关自定义项。

记住遵循 `agent-customization` 指南来创建高效的 instruction。
