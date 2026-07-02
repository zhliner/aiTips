---
name: finishing-a-development-branch
description: 当实现完成、所有测试通过且你需要决定如何整合工作成果时使用——通过提供合并、PR 或清理的结构化选项来引导开发工作的收尾
---

# 完成开发分支

## 概述

通过呈现明确选项并处理所选工作流程来引导开发工作的收尾。

**核心原则：** 验证测试 → 检测环境 → 呈现选项 → 执行选择 → 清理。

**开始时声明：** "我正在使用 finishing-a-development-branch 技能来完成此工作。"

## 流程

### 第 1 步：验证测试

**在呈现选项之前，验证测试是否通过：**

```bash
# 运行项目的测试套件
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：**
```
测试失败（<N> 个失败）。必须在完成之前修复：

[显示失败]

在测试通过之前无法进行合并/PR。
```

停止。不要进入第 2 步。

**如果测试通过：** 继续到第 2 步。

### 第 2 步：检测环境

**在呈现选项之前确定工作空间状态：**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

这决定了显示哪个菜单以及清理如何工作：

| 状态 | 菜单 | 清理 |
|-------|------|---------|
| `GIT_DIR == GIT_COMMON`（普通仓库） | 标准 4 个选项 | 无需清理工作树 |
| `GIT_DIR != GIT_COMMON`，已命名分支 | 标准 4 个选项 | 基于来源（见第 6 步） |
| `GIT_DIR != GIT_COMMON`，分离 HEAD | 减少的 3 个选项（无合并） | 无清理（外部管理） |

### 第 3 步：确定基础分支

```bash
# 尝试常见的基础分支
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

或询问："此分支从 main 分支分出——是否正确？"

### 第 4 步：呈现选项

**普通仓库和已命名分支工作树——精确呈现以下 4 个选项：**

```
实现完成。你想做什么？

1. 在本地合并回 <base-branch>
2. 推送并创建 Pull Request
3. 保持分支原样（我稍后处理）
4. 丢弃此工作成果

选择哪个选项？
```

**分离 HEAD——精确呈现以下 3 个选项：**

```
实现完成。你当前处于分离 HEAD 状态（外部管理的工作空间）。

1. 作为新分支推送并创建 Pull Request
2. 保持原样（我稍后处理）
3. 丢弃此工作成果

选择哪个选项？
```

**不要添加解释**——保持选项简洁。

### 第 5 步：执行选择

#### 选项 1：本地合并

```bash
# 获取主仓库根目录以确保 CWD 安全
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# 先合并——在删除任何内容之前验证成功
git checkout <base-branch>
git pull
git merge <feature-branch>

# 在合并结果上验证测试
<test command>

# 仅在合并成功后：清理工作树（第 6 步），然后删除分支
```

然后：清理工作树（第 6 步），然后删除分支：

```bash
git branch -d <feature-branch>
```

#### 选项 2：推送并创建 PR

```bash
# 推送分支
git push -u origin <feature-branch>
```

**不要清理工作树**——用户需要保留它以便迭代 PR 反馈。

#### 选项 3：保持原样

报告："保持分支 <name>。工作树保留在 <path>。"

**不要清理工作树。**

#### 选项 4：丢弃

**首先确认：**
```
这将永久删除：
- 分支 <name>
- 所有提交：<commit-list>
- 工作树位于 <path>

输入 'discard' 确认。
```

等待精确确认。

如果确认：
```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
```

然后：清理工作树（第 6 步），然后强制删除分支：
```bash
git branch -D <feature-branch>
```

### 第 6 步：清理工作空间

**仅对选项 1 和 4 执行。** 选项 2 和 3 始终保留工作树。

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
WORKTREE_PATH=$(git rev-parse --show-toplevel)
```

**如果 `GIT_DIR == GIT_COMMON`：** 普通仓库，无需清理工作树。完成。

**如果工作树路径在 `.worktrees/` 或 `worktrees/` 下：** Superpowers 创建了此工作树——我们负责清理。

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
git worktree remove "$WORKTREE_PATH"
git worktree prune  # 自愈：清理任何过时的工作树注册
```

**否则：** 宿主环境（harness）拥有此工作空间。不要删除它。如果你的平台提供了工作空间退出工具，使用它。否则，保持工作空间原样。

## 快速参考

| 选项 | 合并 | 推送 | 保留工作树 | 清理分支 |
|--------|-------|------|---------------|----------------|
| 1. 本地合并 | 是 | - | - | 是 |
| 2. 创建 PR | - | 是 | 是 | - |
| 3. 保持原样 | - | - | 是 | - |
| 4. 丢弃 | - | - | - | 是（强制） |

## 常见错误

**跳过测试验证**
- **问题：** 合并了有问题的代码，创建了失败的 PR
- **修复：** 始终在提供选项之前验证测试

**开放式问题**
- **问题：** "我接下来该做什么？" 是模棱两可的
- **修复：** 精确呈现 4 个结构化选项（分离 HEAD 时为 3 个）

**为选项 2 清理工作树**
- **问题：** 删除了用户迭代 PR 所需的工作树
- **修复：** 仅对选项 1 和 4 进行清理

**在删除工作树之前删除分支**
- **问题：** `git branch -d` 因为工作树仍然引用该分支而失败
- **修复：** 先合并，移除工作树，然后删除分支

**从工作树内部运行 git worktree remove**
- **问题：** 当 CWD 位于要删除的工作树内部时，命令静默失败
- **修复：** 在 `git worktree remove` 之前始终 `cd` 到主仓库根目录

**清理 harness 拥有的工作树**
- **问题：** 删除 harness 创建的工作树会导致幻影状态
- **修复：** 仅清理 `.worktrees/` 或 `worktrees/` 下的工作树

**丢弃时不确认**
- **问题：** 意外删除工作成果
- **修复：** 要求输入 "discard" 确认

## 红线

**永远不要：**
- 在测试失败的情况下继续
- 在没有验证合并结果测试的情况下合并
- 未经确认就删除工作成果
- 未经明确请求就强制推送
- 在确认合并成功之前删除工作树
- 清理非你创建的工作树（来源检查）
- 从工作树内部运行 `git worktree remove`

**始终：**
- 在提供选项之前验证测试
- 在呈现菜单之前检测环境
- 精确呈现 4 个选项（分离 HEAD 时为 3 个）
- 为选项 4 获取输入的确认
- 仅清理选项 1 和 4 的工作树
- 在删除工作树之前 `cd` 到主仓库根目录
- 删除后运行 `git worktree prune`
