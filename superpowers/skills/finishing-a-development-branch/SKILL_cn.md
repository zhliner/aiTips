---
name: finishing-a-development-branch
description: 在实现完成、所有测试通过、需要决定如何集成工作时使用——通过提供合并、PR 或清理的结构化选项来引导开发工作的完成
---

# 完成开发分支（Finishing a Development Branch）

## 概述

通过提供清晰的选项并处理所选工作流来引导开发工作的完成。

**核心原则：** 验证测试 → 检测环境 → 展示选项 → 执行选择 → 清理。

**开始时宣布：** "I'm using the finishing-a-development-branch skill to complete this work."

## 流程

### 步骤 1：验证测试

**在展示选项之前，验证测试通过：**

```bash
# 运行项目的测试套件
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：**
```
测试失败（<N> 个失败）。完成前必须修复：

[显示失败信息]

测试通过前无法继续合并/创建 PR。
```

停止。不要继续到步骤 2。

**如果测试通过：** 继续到步骤 2。

### 步骤 2：检测环境

**在展示选项之前确定 workspace 状态：**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

这将决定显示哪个菜单以及清理方式：

| 状态 | 菜单 | 清理 |
|------|------|------|
| `GIT_DIR == GIT_COMMON`（普通 repository） | 标准 4 个选项 | 无需清理 worktree |
| `GIT_DIR != GIT_COMMON`，命名分支 | 标准 4 个选项 | 基于来源（见步骤 6） |
| `GIT_DIR != GIT_COMMON`，detached HEAD | 精简 3 个选项（无合并） | 不清理（外部管理） |

### 步骤 3：确定 base branch

```bash
# 尝试常见的 base branch
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

或者询问："此分支是从 main 分出来的——对吗？"

### 步骤 4：展示选项

**普通 repository 和命名分支 worktree——准确展示以下 4 个选项：**

```
实现完成。你想怎么做？

1. 本地合并回 <base-branch>
2. 推送并创建 Pull Request
3. 保持分支原样（我稍后处理）
4. 丢弃此工作

选择哪个选项？
```

**Detached HEAD——准确展示以下 3 个选项：**

```
实现完成。你当前在 detached HEAD 上（外部管理的 workspace）。

1. 推送为新分支并创建 Pull Request
2. 保持原样（我稍后处理）
3. 丢弃此工作

选择哪个选项？
```

**不要添加解释**——保持选项简洁。

### 步骤 5：执行选择

#### 选项 1：本地合并

```bash
# 获取主 repository 根目录以确保 CWD 安全
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# 先合并——在删除任何东西之前验证成功
git checkout <base-branch>
git pull
git merge <feature-branch>

# 在合并结果上验证测试
<test command>

# 仅在合并成功后：清理 worktree（步骤 6），然后删除分支
```

然后：清理 worktree（步骤 6），再删除分支：

```bash
git branch -d <feature-branch>
```

#### 选项 2：推送并创建 PR

```bash
# 推送分支
git push -u origin <feature-branch>
```

**不要清理 worktree**——用户需要它来根据 PR 反馈进行迭代。

#### 选项 3：保持原样

报告："保留分支 <name>。Worktree 保留在 <path>。"

**不要清理 worktree。**

#### 选项 4：丢弃

**先确认：**
```
这将永久删除：
- 分支 <name>
- 所有 commits：<commit-list>
- Worktree 位于 <path>

输入 'discard' 确认。
```

等待精确确认。

如果确认：
```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
```

然后：清理 worktree（步骤 6），再强制删除分支：
```bash
git branch -D <feature-branch>
```

### 步骤 6：清理工作区

**仅在选项 1 和 4 时执行。** 选项 2 和 3 始终保留 worktree。

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
WORKTREE_PATH=$(git rev-parse --show-toplevel)
```

**如果 `GIT_DIR == GIT_COMMON`：** 普通 repository，无需清理 worktree。完成。

**如果 worktree 路径在 `.worktrees/` 或 `worktrees/` 下：** Superpowers 创建了这个 worktree——我们负责清理。

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
git worktree remove "$WORKTREE_PATH"
git worktree prune  # 自我修复：清理任何过期的注册
```

**否则：** 宿主环境（harness）拥有这个工作区。不要移除它。如果你的平台提供 workspace-exit 工具，使用它。否则，保持工作区不动。

## 快速参考

| 选项 | 合并 | 推送 | 保留 worktree | 清理分支 |
|------|------|------|---------------|----------|
| 1. 本地合并 | 是 | - | - | 是 |
| 2. 创建 PR | - | 是 | 是 | - |
| 3. 保持原样 | - | - | 是 | - |
| 4. 丢弃 | - | - | - | 是（强制） |

## 常见错误

**跳过测试验证**
- **问题：** 合并有问题的代码，创建失败的 PR
- **修复：** 始终在提供选项之前验证测试

**开放式问题**
- **问题：** "What should I do next?" 含义模糊
- **修复：** 准确展示 4 个结构化选项（detached HEAD 时为 3 个）

**为选项 2 清理 worktree**
- **问题：** 删除了用户用于 PR 迭代的 worktree
- **修复：** 仅在选项 1 和 4 时清理

**在移除 worktree 之前删除分支**
- **问题：** `git branch -d` 失败，因为 worktree 仍然引用该分支
- **修复：** 先合并，移除 worktree，然后删除分支

**从 worktree 内部运行 git worktree remove**
- **问题：** 当 CWD 在被移除的 worktree 内部时，命令静默失败
- **修复：** 始终先 `cd` 到主 repository 根目录，再执行 `git worktree remove`

**清理 harness 拥有的 worktrees**
- **问题：** 移除 harness 创建的 worktree 会导致幻影状态
- **修复：** 仅清理 `.worktrees/` 或 `worktrees/` 下的 worktrees

**丢弃时不确认**
- **问题：** 意外删除工作成果
- **修复：** 要求输入 "discard" 确认

## 危险信号

**绝不：**
- 在测试失败时继续
- 在未验证合并结果测试的情况下合并
- 未经确认就删除工作成果
- 未经明确要求就 force-push
- 在确认合并成功之前移除 worktree
- 清理不是你创建的 worktrees（来源检查）
- 从 worktree 内部运行 `git worktree remove`

**始终：**
- 在提供选项之前验证测试
- 在展示菜单之前检测环境
- 准确展示 4 个选项（detached HEAD 时为 3 个）
- 为选项 4 获取输入确认
- 仅在选项 1 和 4 时清理 worktree
- 在移除 worktree 之前 `cd` 到主 repository 根目录
- 移除后运行 `git worktree prune`
