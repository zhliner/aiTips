# Pi 工具映射

Skill 以动作形式表达（"派发 subagent"、"创建 todo"、"读取文件"）。在 Pi 上，这些对应到以下工具。

| Skill 请求的动作 | Pi 对应工具 |
| --- | --- |
| 派发 subagent（`Subagent (general-purpose):` 模板） | 如果有可用的已安装 subagent 工具（如来自 `pi-subagents` 的 `subagent`），则使用它 |
| 任务跟踪（"创建 todo"、"标记完成"） | 如果有可用的已安装 todo/task 工具则使用它，否则在计划或 `TODO.md` 中跟踪任务 |

## Subagents

Pi 核心不附带标准 subagent 工具。`pi-subagents` 包是一个强烈推荐的可选伴侣，提供具有单 agent、链式、并行、异步、分叉上下文以及恢复/状态工作流的 `subagent` 工具。如果没有可用的 subagent 工具，不要伪造 `Task` 调用；在当前会话中顺序执行，或说明可选的 subagent 功能未安装。

## 任务列表

Pi 核心不附带标准任务列表工具。如果安装了 todo/task 扩展，使用其文档化的工具。否则使用 Superpowers 计划文件、Markdown 检查清单或仓库本地的 `TODO.md` 进行任务跟踪。较旧的 Superpowers 文档可能引用 `TodoWrite`；将其视为上述任务跟踪动作。
