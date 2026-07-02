---
name: using-git-worktrees
description: 在开始需要与当前工作空间隔离的功能工作时使用，或在执行实现计划之前使用——通过原生工具或 git worktree 回退方案确保隔离工作空间存在
---

# 使用 Git Worktrees

## 概述

确保工作在隔离的工作空间中进行。优先使用你的平台的原生 worktree 工具。仅在没有原生工具可用时才回退到手动 git worktree。

**核心原则：** 首先检测已存在的隔离环境。然后使用原生工具。然后回退到 git。绝不要与工具框架对抗。

**开始时声明：** "我正在使用 using-git-worktrees skill 设置隔离工作空间。"

## 步骤 0：检测已存在的隔离环境

**在创建任何东西之前，先检查你是否已经处于一个隔离的工作空间中。**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**子模块检查：** `GIT_DIR != GIT_COMMON` 在 git 子模块内部也为真。在得出"已经在 worktree 中"的结论之前，验证你不在子模块中：

```bash
# 如果返回一个路径，你在子模块中，而非 worktree —— 视为普通仓库
git rev-parse --show-superproject-working-tree 2>/dev/null
```

**如果 `GIT_DIR != GIT_COMMON`（且不在子模块中）：** 你已经在一个链接 worktree 中。跳到步骤 2（项目设置）。不要再创建另一个 worktree。

根据分支状态报告：
- 在分支上："已经在 `<path>` 的隔离工作空间中，位于分支 `<name>`。"
- Detached HEAD："已经在 `<path>` 的隔离工作空间中（detached HEAD，外部管理）。完成时需要创建分支。"

**如果 `GIT_DIR == GIT_COMMON`（或在子模块中）：** 你处于普通的仓库检出中。

用户是否已在指令中表明了 worktree 偏好？如果没有，在创建 worktree 之前征得同意：

> "要我设置一个隔离的 worktree 吗？它可以保护你的当前分支不受变更影响。"

尊重任何已声明的偏好，无需再次询问。如果用户拒绝同意，在原位工作并跳到步骤 2。

## 步骤 1：创建隔离工作空间

**你有两种机制可用。按此顺序尝试。**

### 1a. 原生 Worktree 工具（首选）

用户已请求隔离工作空间（步骤 0 同意）。你是否已有创建 worktree 的方式？它可能是一个名为 `EnterWorktree`、`WorktreeCreate` 的工具、一个 `/worktree` 命令或 `--worktree` 标志。如果有，使用它并跳到步骤 2。

原生工具会自动处理目录位置、分支创建和清理。当你拥有原生工具时使用 `git worktree add` 会创建你的工具框架无法看到或管理的幽灵状态。

仅在没有任何原生 worktree 工具可用时才继续到步骤 1b。

### 1b. Git Worktree 回退方案

**仅在步骤 1a 不适用时使用**——你没有可用的原生 worktree 工具。使用 git 手动创建 worktree。

#### 目录选择

按此优先级顺序。显式的用户偏好总是优先于观察到的文件系统状态。

1. **检查你的指令中是否有声明的 worktree 目录偏好。** 如果用户已指定，直接使用，无需询问。

2. **检查是否存在现有的项目本地 worktree 目录：**
   ```bash
   ls -d .worktrees 2>/dev/null     # 首选（隐藏目录）
   ls -d worktrees 2>/dev/null      # 替代
   ```
   如果找到，使用它。如果两者都存在，`.worktrees` 优先。

3. **如果没有其他指引可用**，默认使用项目根目录下的 `.worktrees/`。

#### 安全检查（仅项目本地目录）

**创建 worktree 之前必须验证目录是 git 忽略的：**

```bash
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**如果未被忽略：** 添加到 .gitignore，提交该变更，然后继续。

**为什么至关重要：** 防止意外将 worktree 内容提交到仓库。

#### 创建 Worktree

```bash
# 基于所选位置确定路径
path="$LOCATION/$BRANCH_NAME"

git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

**沙箱回退：** 如果 `git worktree add` 因权限错误失败（沙箱拒绝），告诉用户沙箱阻止了 worktree 创建，你将在当前目录中工作。然后在原位运行设置和基线测试。

## 步骤 2：项目设置

自动检测并运行适当的设置：

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

运行测试以确保工作空间初始状态干净：

```bash
# 使用项目适当的命令
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：** 报告失败，询问是要继续还是先调查。

**如果测试通过：** 报告就绪。

### 报告

```
Worktree 就绪于 <full-path>
测试通过 (<N> 项测试，0 项失败)
准备实现 <feature-name>
```

## 快速参考

| 情况 | 动作 |
|-----------|--------|
| 已经在链接 worktree 中 | 跳过创建（步骤 0） |
| 在子模块中 | 视为普通仓库（步骤 0 检查） |
| 有原生 worktree 工具可用 | 使用它（步骤 1a） |
| 没有原生工具 | Git worktree 回退（步骤 1b） |
| `.worktrees/` 存在 | 使用它（验证是否被忽略） |
| `worktrees/` 存在 | 使用它（验证是否被忽略） |
| 两者都存在 | 使用 `.worktrees/` |
| 两者都不存在 | 检查指令文件，然后默认使用 `.worktrees/` |
| 目录未被忽略 | 添加到 .gitignore + 提交 |
| 创建时权限错误 | 沙箱回退，原位工作 |
| 基线时测试失败 | 报告失败 + 询问 |
| 无 package.json/Cargo.toml | 跳过依赖安装 |

## 常见错误

### 与工具框架对抗

- **问题：** 在平台已经提供隔离环境时仍使用 `git worktree add`
- **修复：** 步骤 0 检测已存在的隔离环境。步骤 1a 优先使用原生工具。

### 跳过检测

- **问题：** 在已有的 worktree 内创建嵌套的 worktree
- **修复：** 在创建任何东西之前始终运行步骤 0

### 跳过忽略验证

- **问题：** Worktree 内容被跟踪，污染 git status
- **修复：** 在创建项目本地 worktree 之前始终使用 `git check-ignore`

### 假设目录位置

- **问题：** 造成不一致，违反项目约定
- **修复：** 按优先级：显式指令 > 已存在的项目本地目录 > 默认

### 在测试失败时继续

- **问题：** 无法区分新 bug 和已存在的问题
- **修复：** 报告失败，获得显式许可才能继续

## 红线

**绝不：**
- 当步骤 0 检测到已有隔离环境时创建 worktree
- 当你有原生 worktree 工具（如 `EnterWorktree`）时使用 `git worktree add`。这是第一号错误——如果你有它，就使用它。
- 跳过步骤 1a 直接跳到步骤 1b 的 git 命令
- 在未验证目录是否被忽略的情况下创建 worktree（项目本地）
- 跳过基线测试验证
- 在未询问的情况下在测试失败时继续

**始终：**
- 先运行步骤 0 检测
- 优先使用原生工具而非 git 回退
- 遵循目录优先级：显式指令 > 已存在的项目本地目录 > 默认
- 验证项目本地目录是否被忽略
- 自动检测并运行项目设置
- 验证干净的测试基线
