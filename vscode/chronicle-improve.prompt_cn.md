---
name: chronicle:improve
description: 根据会话历史中的摩擦模式改进 agent 指令
---
分析我最近的聊天会话历史中的摩擦模式，并建议改进我的 agent 指令文件。使用 **chronicle** skill —— 它记录了 `copilot_sessionStoreSql` 工具、session-store schema，以及 Improve 工作流，用于检测跨会话的重复失败、用户纠正和反复出现的摩擦，然后提出基于数据的改进建议添加到项目的 agent 指令中。

当你调用 `copilot_sessionStoreSql` 时，每次调用都设置 `subcommand: "improve"`。
