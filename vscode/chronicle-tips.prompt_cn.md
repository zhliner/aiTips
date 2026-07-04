---
name: chronicle:tips
description: 根据你的聊天会话使用模式获取个性化建议
---
分析我最近的聊天会话历史，并给出个性化建议以改进我的工作流。使用 **chronicle** skill —— 它记录了 `copilot_sessionStoreSql` 工具、session-store schema，以及用于从 `sessions`、`turns`、`session_files` 和 `session_refs` 中调查使用模式的 Tips 工作流。

当你调用 `copilot_sessionStoreSql` 时，每次调用都设置 `subcommand: "tips"`。
