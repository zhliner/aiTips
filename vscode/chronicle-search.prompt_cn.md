---
name: chronicle:search
description: 按关键词、文件路径或 PR/issue 引用搜索最近的聊天会话
---
在我的 Copilot 会话历史中搜索我提供的查询内容 —— 关键词、文件路径或 PR/issue/commit 引用 —— 并列出匹配的会话。使用 **chronicle** skill —— 它记录了 `copilot_sessionStoreSql` 工具、session-store schema（`sessions` 表的主键是 `id`；对话内容存储在 `turns` 中，而非 `sessions` 上；在本地 SQLite 中使用 FTS5 `search_index` 表并直接 select `session_id` —— 永远不要将 `search_index.rowid` 与 `turns.rowid` 进行 join），以及包含云端性能规则的 Search 工作流（通过 `WITH hits ... JOIN sessions` 进行一次性聚合，默认对 `turns` 使用 90 天窗口）。

当你调用 `copilot_sessionStoreSql` 时，每次调用都设置 `subcommand: "search"`。
