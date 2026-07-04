---
name: get-search-view-results
description: '获取 VS Code 中 Search 视图的当前搜索结果'
---

# 获取 Search 视图结果

1. VS Code 有一个 Search 视图，它可能包含已有的搜索结果。
2. 要获取当前搜索结果，可以使用 VS Code 命令 `search.action.getSearchResults`。
3. 通过 `copilot_runVscodeCommand` 工具运行该命令。确保将 `skipCheck` 参数传递为 true，以避免检查命令是否存在，因为我们知道它存在。
