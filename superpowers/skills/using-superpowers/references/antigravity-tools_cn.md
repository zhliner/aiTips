# Antigravity CLI (`agy`) 工具映射

Skills 使用动作来表达（"dispatch a subagent"、"create a todo"、"read a file"）。在 Antigravity CLI（`agy`）上，这些动作映射为以下工具。

| Skills 请求的动作 | Antigravity CLI 等价工具 |
|----------------------|----------------------|
| 派遣 subagent（`Subagent (general-purpose):` 模板） | `invoke_subagent` 配合内置 `TypeName`——`self` 用于全功能工作，`research` 用于只读（参见 [Subagent 支持](#subagent-支持)） |
| 任务跟踪（"create a todo"、"mark complete"） | **task artifact**——`write_to_file` 配合 `IsArtifact: true` 和 `ArtifactType: "task"`（参见 [任务跟踪](#任务跟踪)）。**不是** `manage_task`，后者管理后台进程。 |

## 任务跟踪

Antigravity **没有 todo 工具**（`manage_task` 管理后台进程——`list`/`kill`/`status`/`send_input`——它*不是*检查清单）。当 skill 要求创建 todo 列表或跟踪任务时，维护一个 **task artifact**：一个使用 `write_to_file`（`IsArtifact: true`、`ArtifactMetadata.ArtifactType: "task"`）保存的 markdown 检查清单，随着进展使用 `replace_file_content` / `multi_replace_file_content` 进行编辑。

在任何多步骤任务的开始时，创建 task artifact 并列出计划中的每个步骤。完成每个步骤后，编辑 artifact 将其标记为完成（`- [x]`）。如果计划变更，更新检查清单。保持其最新——它是剩余工作的唯一真实来源；一旦对话变长，在开始每个步骤之前重新读取它。
