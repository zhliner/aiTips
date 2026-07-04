---
name: chronicle:reindex
description: 重建本地会话索引并同步到云端
---
重新索引我的 session store 以获取任何缺失的会话。添加 'force' 以重新处理已索引的会话。

当你调用 `copilot_sessionStoreSql` 时，设置 `subcommand: "reindex"`。
