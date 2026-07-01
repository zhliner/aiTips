---
name: using-git-worktrees
description: 当开始需要与当前工作区隔离的功能开发时，或在执行实现计划之前使用 — 通过原生工具或 git worktree 回退方案确保存在隔离的工作区
---

# 使用 Git Worktrees

## 概述

确保工作在隔离的工作区中进行。优先使用平台的原生 worktree 工具。仅在没有原生工具可用时才回退到手动 git worktree。

**核心原则：** 先检测已有的隔离。然后使用原生工具。然后回退到 git。永远不要对抗工具框架。

**开始时的宣告：** "我正在使用 using-git-worktrees 技能来设置隔离的工作区。"

## 步骤 0：检测已有的隔离

**在创建任何东西之前，检查你是否已经处于隔离的工作区中。**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**子模块守卫：** `GIT_DIR != GIT_COMMON` 在 git 子模块内部也为真。在得出"已经在 worktree 中"的结论之前，验证你不在子模块中：

```bash
# 如果此命令返回路径，说明你在子模块中，而非 worktree — 按普通仓库处理
git rev-parse --show-superproject-working-tree 2>/dev/null
```

**如果 `GIT_DIR != GIT_COMMON`（且不是子模块）：** 你已经在一个关联的 worktree 中。跳至步骤 2（项目设置）。不要创建另一个 worktree。

报告分支状态：
- 在分支上："已在 `<path>` 的隔离工作区中，位于 `<name>` 分支。"
- 分离 HEAD："已在 `<path>` 的隔离工作区中（分离 HEAD，外部管理）。完成时需要创建分支。"

**如果 `GIT_DIR == GIT_COMMON`（或在子模块中）：** 你处于普通的仓库检出中。

用户是否已在其指示中说明了 worktree 偏好？如果没有，在创建 worktree 之前请求同意：

> "你希望我设置一个隔离的 worktree 吗？它可以保护你当前分支不受更改影响。"

如果已有声明的偏好则直接遵循，不要询问。如果用户拒绝同意，则在当前位置工作并跳至步骤 2。

## 步骤 1：创建隔离工作区

**你有两种机制。按此顺序尝试。**

### 1a. 原生 Worktree 工具（首选）

用户已经请求了隔离工作区（步骤 0 中已获同意）。你是否已经有创建 worktree 的方式？它可能是一个名为 `EnterWorktree`、`WorktreeCreate` 的工具，一个 `/worktree` 命令，或一个 `--worktree` 标志。如果有，使用它并跳至步骤 2。

原生工具自动处理目录放置、分支创建和清理。当你拥有原生工具时使用 `git worktree add` 会创建工具框架无法看到或管理的幽灵状态。

只有当没有可用的原生 worktree 工具时，才进入步骤 1b。

### 1b. Git Worktree 回退方案

**仅当步骤 1a 不适用时使用此方案** — 你没有可用的原生 worktree 工具。使用 git 手动创建 worktree。

#### 目录选择

按以下优先级顺序。用户的显式偏好始终胜过观察到的文件系统状态。

1. **检查你的指示中是否有声明的 worktree 目录偏好。** 如果用户已经指定了，直接使用，不要询问。

2. **检查是否存在项目本地的 worktree 目录：**
   ```bash
   ls -d .worktrees 2>/dev/null     # 首选（隐藏）
   ls -d worktrees 2>/dev/null      # 备选
   ```
   如果找到，使用它。如果两者都存在，`.worktrees` 优先。

3. **如果没有其他指导可用**，默认使用项目根目录下的 `.worktrees/`。

#### 安全验证（仅项目本地目录）

**在创建 worktree 之前必须验证目录已被忽略：**

```bash
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**如果未被忽略：** 添加到 `.gitignore`，提交变更，然后继续。

**为什么关键：** 防止意外将 worktree 内容提交到仓库。

#### 创建 Worktree

```bash
# 根据选择的路径确定路径
path="$LOCATION/$BRANCH_NAME"

git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

**沙箱回退：** 如果 `git worktree add` 因权限错误失败（沙箱拒绝），告诉用户沙箱阻止了 worktree 创建，你改为在当前目录中工作。然后在当前位置运行设置和基线测试。

## 步骤 2：项目设置

自动检测并运行相应的设置：

```bash
# Node.js
if [ -f package.json ]; then npm install; fi

# Rust
if [ -f Cargo.toml ]; then cargo build; fi

# Python
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi

# Go
if [ -f go.mod ]; then go mod download; fi
```

## 步骤 3：验证干净的基线

运行测试以确保工作区从干净状态开始：

```bash
# 使用项目合适的命令
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：** 报告失败，询问是否继续还是调查。

**如果测试通过：** 报告就绪。

### 报告

```
Worktree 就绪，位于 <full-path>
测试通过（<N> 个测试，0 个失败）
准备实现 <feature-name>
```

## 快速参考

| 情况 | 行动 |
|-----------|--------|
| 已在关联 worktree 中 | 跳过创建（步骤 0） |
| 在子模块中 | 按普通仓库处理（步骤 0 守卫） |
| 原生 worktree 工具可用 | 使用它（步骤 1a） |
| 无原生工具 | Git worktree 回退（步骤 1b） |
| `.worktrees/` 存在 | 使用它（验证已忽略） |
| `worktrees/` 存在 | 使用它（验证已忽略） |
| 两者都存在 | 使用 `.worktrees/` |
| 两者都不存在 | 检查指示文件，然后默认 `.worktrees/` |
| 目录未被忽略 | 添加到 `.gitignore` + 提交 |
| 创建时权限错误 | 沙箱回退，在当前位置工作 |
| 基线测试失败 | 报告失败 + 询问 |
| 无 package.json/Cargo.toml | 跳过依赖安装 |

## 常见错误

### 对抗工具框架

- **问题：** 在平台已经提供隔离的情况下使用 `git worktree add`
- **修复：** 步骤 0 检测已有的隔离。步骤 1a 遵从原生工具。

### 跳过检测

- **问题：** 在已经存在的 worktree 内部创建嵌套 worktree
- **修复：** 始终在创建任何东西之前先运行步骤 0

### 跳过忽略验证

- **问题：** worktree 内容被跟踪，污染 git status
- **修复：** 在创建项目本地 worktree 之前始终使用 `git check-ignore`

### 假设目录位置

- **问题：** 造成不一致，违反项目约定
- **修复：** 遵循优先级：显式指示 > 已有项目本地目录 > 默认

### 在测试失败时继续

- **问题：** 无法区分新 bug 和已存在的问题
- **修复：** 报告失败，获得明确许可后再继续

## 红色警报

**绝不要：**
- 在步骤 0 检测到已有隔离时创建 worktree
- 当你拥有原生 worktree 工具（如 `EnterWorktree`）时使用 `git worktree add`。这是第一号错误 — 如果你有它，就用它。
- 跳过步骤 1a 直接跳到步骤 1b 的 git 命令
- 未验证被忽略就创建 worktree（项目本地）
- 跳过基线测试验证
- 在测试失败时未经询问就继续

**始终：**
- 先运行步骤 0 检测
- 优先使用原生工具而非 git 回退
- 遵循目录优先级：显式指示 > 已有项目本地目录 > 默认
- 对项目本地目录验证目录已被忽略
- 自动检测并运行项目设置
- 验证干净的测试基线
