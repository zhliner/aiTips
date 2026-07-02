# Pi 工具映射

技能使用操作语言表达（"分派一个 subagent""创建一个 todo""读取一个文件"）。在 Pi 上，这些操作对应以下工具。

| 技能请求的操作 | Pi 等价工具 |
| --- | --- |
| 分派一个 subagent（`Subagent (general-purpose):` 模板） | 使用已安装的 subagent 工具，例如 `pi-subagents` 中的 `subagent`（如果可用） |
| 任务跟踪（"创建一个 todo""标记完成"） | 使用已安装的 todo/任务工具（如果可用），否则在计划或 `TODO.md` 中跟踪任务 |

## Subagent

Pi 核心不提供标准 subagent 工具。`pi-subagents` 包是一个强力可选伴侣，提供带有单 agent、链式、并行、异步、forked-context 和 resume/status 工作流的 `subagent` 工具。如果没有可用的 subagent 工具，不要伪造 `Task` 调用；在当前会话中顺序执行，或说明可选的 subagent 能力未安装。

## 任务列表

Pi 核心不提供标准任务列表工具。如果安装了 todo/任务扩展，使用其文档化的工具。否则使用 Superpowers 计划文件、Markdown 清单或仓库本地的 `TODO.md` 进行任务跟踪。较旧的 Superpowers 文档可能引用 `TodoWrite`；请将其视为上述任务跟踪操作。
