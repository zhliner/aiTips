---
name: chronicle:cost-tips
description: 获取个性化建议以减少 token 用量和 Copilot 成本
---
分析我最近的聊天会话历史，并给出个性化的、基于数据的建议，以减少 token 用量和 Copilot 成本。使用 **chronicle** skill —— 它记录了 `copilot_sessionStoreSql` 工具、session-store schema，以及用于发现高开销会话、token 密集型模式和具体习惯改进的 Cost Tips 工作流。

当你调用 `copilot_sessionStoreSql` 时，每次调用都设置 `subcommand: "cost-tips"`。
