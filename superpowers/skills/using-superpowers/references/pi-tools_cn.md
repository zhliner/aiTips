# Pi 工具映射

Skills 使用动作来表达（"dispatch a subagent"、"create a todo"、"read a file"）。在 Pi 上，这些动作映射为以下工具。

| Skills 请求的动作 | Pi 等价工具 |
| --- | --- |
| 派遣 subagent（`Subagent (general-purpose):` 模板） | 使用已安装的 subagent 工具，如 `pi-subagents` 中的 `subagent`（如果可用） |
| 任务跟踪（"create a todo"、"mark complete"） | 使用已安装的 todo/task 工具（如果可用），否则在计划或 `TODO.md` 中跟踪任务 |

## Subagents

Pi 核心不提供标准的 subagent 工具。`pi-subagents` 包是一个强力的可选配套组件，提供 `subagent` 工具，支持单 agent、链式、并行、异步、分叉上下文和恢复/状态工作流。如果没有可用的 subagent 工具，不要伪造 `Task` 调用；在当前会话中顺序执行，或说明可选的 subagent 功能未安装。

## 任务列表

Pi 核心不提供标准的任务列表工具。如果安装了 todo/task 扩展，使用其文档中说明的工具。否则使用 Superpowers 计划文件、Markdown 中的检查清单或 repository 本地的 `TODO.md` 进行任务跟踪。较旧的 Superpowers 文档可能提到 `TodoWrite`；将其视为上述任务跟踪动作。
