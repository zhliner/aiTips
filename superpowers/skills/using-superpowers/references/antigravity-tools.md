# Antigravity CLI (`agy`) 工具映射

Skill 以动作形式表达（"派发 subagent"、"创建 todo"、"读取文件"）。在 Antigravity CLI (`agy`) 上，这些对应到以下工具。

| Skill 请求的动作 | Antigravity CLI 对应工具 |
|----------------------|----------------------|
| 派发 subagent（`Subagent (general-purpose):` 模板） | `invoke_subagent` 配合内置的 `TypeName`——`self` 用于全能力工作，`research` 用于只读（参见 [Subagent 支持](#subagent-支持)） |
| 任务跟踪（"创建 todo"、"标记完成"） | 一个 **task artifact**——使用 `write_to_file` 配合 `IsArtifact: true` 和 `ArtifactType: "task"`（参见 [任务跟踪](#任务跟踪)）。**不要**使用 `manage_task`，它是用来管理后台进程的。 |

## 任务跟踪

Antigravity **没有** todo 工具（`manage_task` 管理后台进程——`list`/`kill`/`status`/`send_input`——它**不是**检查清单）。当 skill 要求创建 todo 列表或跟踪任务时，维护一个 **task artifact**：一个 markdown 检查清单，使用 `write_to_file`（`IsArtifact: true`，`ArtifactMetadata.ArtifactType: "task"`）保存，在过程中使用 `replace_file_content` / `multi_replace_file_content` 编辑。

在任何多步骤任务开始时，创建 task artifact，列出计划的每个步骤。每完成一个步骤，编辑 artifact 将其标记为完成（`- [x]`）。如果计划变更，更新检查清单。保持其最新——它是你还需做什么的唯一事实来源；当对话变长时，在每个步骤开始前重新阅读它。
