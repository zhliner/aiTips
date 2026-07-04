---
name: chronicle:standup
description: 根据最近的聊天会话生成站会报告
---
根据我最近的编码会话生成一份站会报告。使用 **chronicle** skill —— 它记录了 `copilot_sessionStoreSql` 工具、session-store schema，以及用于汇总 `sessions`、`session_refs`、`turns` 和 `session_files` 中过去 24 小时活动的 Standup 工作流。

当你调用 `copilot_sessionStoreSql` 时，每次调用都设置 `subcommand: "standup"`。
