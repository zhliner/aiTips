---
name: create-hook
description: '创建一个 hook（.json）以强制执行策略或自动化 agent 生命周期事件。'
argument-hint: 需要强制执行或自动化什么？
disable-model-invocation: true
---
相关 skill：`agent-customization`。加载并遵循 **hooks.md** 中的模板和原则。

引导用户在 `.github/hooks/` 中创建一个 hook。

## 从对话中提取
首先，回顾对话历史。如果用户一直在表达对 agent 行为的担忧（例如，"不要运行这个命令"、"在做 X 之前总是检查"、"注入这个上下文"），将其泛化为一个 hook。提取：
- 应被阻止或门控的操作
- 在特定点应注入的上下文
- 在会话开始/结束或工具使用时的自动化需求

## 必要时澄清
如果对话中没有明确出现策略需求，进行澄清：
- 什么事件应触发此 hook？（例如 PreToolUse、SessionStart、Stop）
- 它应阻止、警告还是注入上下文？
- 是否需要配套的脚本？

## 路径约定
- Hook 命令和配套脚本的 `cwd` 默认为 workspace 根目录（当 workspace 文件夹可用时）；否则 `cwd` 回退到用户主目录。相对路径从 `cwd` 解析。绝对路径在有意使用时是可以的——只需明确你使用的是哪种。
- 不要使用环境变量进行路径设置。唯一的例外是作为 agent 插件一部分发布的 hook 中由插件提供的环境变量；在插件上下文之外，绝不使用环境变量进行路径设置。

## 迭代
1. 起草 hook JSON（及任何脚本）并保存。
2. 找出最模糊或最薄弱的部分，并就此提问。
3. 确定后，总结 hook 强制执行的内容，建议测试方法，并提出接下来要创建的相关自定义项。

记住遵循 `agent-customization` 指南来创建高效的 hook。
