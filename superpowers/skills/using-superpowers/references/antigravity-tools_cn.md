# Antigravity CLI（`agy`）工具映射

技能使用操作语言表达（"分派一个 subagent""创建一个 todo""读取一个文件"）。在 Antigravity CLI（`agy`）上，这些操作对应以下工具。

| 技能请求的操作 | Antigravity CLI 等价工具 |
|----------------------|----------------------|
| 分派一个 subagent（`Subagent (general-purpose):` 模板） | 使用内置 `TypeName` 调用 `invoke_subagent` —— `self` 用于全能力工作，`research` 用于只读操作（参见[Subagent 支持](#subagent-支持)） |
| 任务跟踪（"创建一个 todo""标记完成"） | **任务 artifact** —— 使用 `write_to_file` 并设置 `IsArtifact: true` 和 `ArtifactType: "task"`（参见[任务跟踪](#任务跟踪)）。**不是** `manage_task`，后者管理后台进程。 |

## 任务跟踪

Antigravity **没有 todo 工具**（`manage_task` 管理后台进程 —— `list`/`kill`/`status`/`send_input` —— 它**不是**清单工具）。当技能要求创建 todo 列表或跟踪任务时，维护一个**任务 artifact**：使用 `write_to_file`（`IsArtifact: true`，`ArtifactMetadata.ArtifactType: "task"`）保存一个 markdown 清单，并在执行过程中使用 `replace_file_content` / `multi_replace_file_content` 进行编辑。

在任何多步骤任务开始时，创建任务 artifact，列出计划中的每一步。完成每一步后，编辑 artifact 将其标记为已完成（`- [x]`）。如果计划有变，更新清单。保持其时效性 —— 它是你还剩什么工作的唯一事实来源；当对话变长时，在开始每一步之前重新读取它。
