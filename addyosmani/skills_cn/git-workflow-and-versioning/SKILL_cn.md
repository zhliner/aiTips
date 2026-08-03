---
name: git-workflow-and-versioning
description: 规范 git 工作流实践。在任何代码变更时使用。在提交、创建分支、解决冲突，或需要组织多个并行工作流时使用。
---

# Git 工作流与版本控制（Git Workflow and Versioning）

## 概述

Git 是你的安全网。将 commit 视为存档点，branch 视为沙盒，history 视为文档。在 AI agent 高速生成代码的时代，严格的版本控制是保持变更可管理、可审查、可回退的机制。

## 何时使用

始终使用。每一次代码变更都通过 git 流转。

## 核心原则

### 基于主干的开发（推荐）

保持 `main` 始终可部署。在短生命周期的 feature branch 中工作，并在 1-3 天内合并回主干。长期存在的开发分支是隐性成本——它们会产生分歧、引发合并冲突，并延迟集成。DORA 的研究一致表明，基于主干的开发与高绩效工程团队正相关。

```
main ──●──●──●──●──●──●──●──●──●──  (始终可部署)
        ╲      ╱  ╲    ╱
         ●──●─╱    ●──╱    ← 短生命周期的 feature branch（1-3 天）
```

这是推荐的默认方式。使用 gitflow 或长期分支的团队可以将这些原则（原子 commit、小变更、描述性消息）适配到他们的分支模型中——commit 纪律比具体的分支策略更重要。

- **开发分支是成本。** 分支每多存在一天，合并风险就增加一分。
- **发布分支是可以接受的。** 当你需要在 main 继续推进的同时稳定一个发布版本时。
- **Feature flag > 长期分支。** 优先将未完成的工作部署在 flag 后面，而不是在分支上保留数周。

### 1. 尽早提交，频繁提交

每个成功的增量都应有自己的 commit。不要积累大量未提交的变更。

```
工作模式：
  实现切片 → 测试 → 验证 → 提交 → 下一个切片

不要这样：
  实现所有功能 → 祈祷能跑 → 巨大的一次提交
```

commit 就是存档点。如果下一次改动出了问题，你可以立即回退到上一个已知的正确状态。

### 2. 原子 Commit

每个 commit 只做一件逻辑上的事情：

```
# 好的做法：每个 commit 自成一体
git log --oneline
a1b2c3d Add task creation endpoint with validation
d4e5f6g Add task creation form component
h7i8j9k Connect form to API and add loading state
m1n2o3p Add task creation tests (unit + integration)

# 不好的做法：所有东西混在一起
git log --oneline
x1y2z3a Add task feature, fix sidebar, update deps, refactor utils
```

### 3. 描述性的消息

commit message 解释的是*为什么*，而不仅仅是*做了什么*：

```
# 好的做法：解释了意图
feat: add email validation to registration endpoint

Prevents invalid email formats from reaching the database.
Uses Zod schema validation at the route handler level,
consistent with existing validation patterns in auth.ts.

# 不好的做法：描述了从 diff 就能看出的内容
update auth.ts
```

**格式：**
```
<type>: <short description>

<optional body explaining why, not what>
```

**类型：**
- `feat` — 新功能
- `fix` — 缺陷修复
- `refactor` — 既不修复缺陷也不新增功能的代码变更
- `test` — 添加或更新测试
- `docs` — 仅文档变更
- `chore` — 工具链、依赖、配置

### 4. 保持关注点分离

不要将格式化变更与行为变更混合。不要将重构与功能变更混合。每种类型的变更应该是独立的 commit——理想情况下也是独立的 PR：

```
# 好的做法：关注点分离
git commit -m "refactor: extract validation logic to shared utility"
git commit -m "feat: add phone number validation to registration"

# 不好的做法：关注点混合
git commit -m "refactor validation and add phone number field"
```

**将重构与功能开发分开。** 重构变更和功能变更是两种不同的变更——应分别提交。这使每个变更更易于审查、回退和在历史中理解。小的清理工作（如重命名变量）可以在审查者的判断下包含在功能 commit 中。

### 5. 控制变更规模

目标为每个 commit/PR 约 100 行。超过约 1000 行的变更应该拆分。有关如何拆解大型变更的方法，请参阅 `code-review-and-quality` 中的拆分策略。

```
~100 行   → 易于审查，易于回退
~300 行   → 对于单个逻辑变更可以接受
~1000 行  → 拆分为更小的变更
```

## 分支策略

### Feature Branch

```
main（始终可部署）
  │
  ├── feature/task-creation    ← 每个分支一个功能
  ├── feature/user-settings    ← 并行开发
  └── fix/duplicate-tasks      ← 缺陷修复
```

- 从 `main`（或团队的默认分支）创建分支
- 保持分支短生命周期（在 1-3 天内合并）——长期分支是隐性成本
- 合并后删除分支
- 对于未完成的功能，优先使用 feature flag 而非长期分支

### 分支命名

```
feature/<short-description>   → feature/task-creation
fix/<short-description>       → fix/duplicate-tasks
chore/<short-description>     → chore/update-deps
refactor/<short-description>  → refactor/auth-module
```

## 使用 Worktree

对于并行的 AI agent 工作，使用 git worktree 同时运行多个分支：

```bash
# 为 feature branch 创建 worktree
git worktree add ../project-feature-a feature/task-creation
git worktree add ../project-feature-b feature/user-settings

# 每个 worktree 是一个独立的目录，拥有自己的分支
# Agent 可以并行工作而互不干扰
ls ../
  project/              ← main 分支
  project-feature-a/    ← task-creation 分支
  project-feature-b/    ← user-settings 分支

# 完成后，合并并清理
git worktree remove ../project-feature-a
```

优势：
- 多个 agent 可以同时开发不同的功能
- 无需切换分支（每个目录有自己的分支）
- 如果某个实验失败，删除 worktree 即可——不会丢失任何内容
- 变更在显式合并之前保持隔离

## 存档点模式

```
Agent 开始工作
    │
    ├── 做一个变更
    │   ├── 测试通过？ → 提交 → 继续
    │   └── 测试失败？ → 回退到上一次 commit → 排查
    │
    ├── 做另一个变更
    │   ├── 测试通过？ → 提交 → 继续
    │   └── 测试失败？ → 回退到上一次 commit → 排查
    │
    └── 功能完成 → 所有 commit 构成清晰的历史记录
```

这种模式意味着你永远不会丢失超过一个增量的工作。如果 agent 偏离了正轨，`git reset --hard HEAD` 可以让你回到上一个成功的状态。

## 变更摘要

在任何修改之后，提供结构化的摘要。这使审查更容易，记录范围纪律，并暴露非预期的变更：

```
CHANGES MADE:
- src/routes/tasks.ts: Added validation middleware to POST endpoint
- src/lib/validation.ts: Added TaskCreateSchema using Zod

THINGS I DIDN'T TOUCH (intentionally):
- src/routes/auth.ts: Has similar validation gap but out of scope
- src/middleware/error.ts: Error format could be improved (separate task)

POTENTIAL CONCERNS:
- The Zod schema is strict — rejects extra fields. Confirm this is desired.
- Added zod as a dependency (72KB gzipped) — already in package.json
```

这种模式可以尽早发现错误的假设，并为审查者提供清晰的变更地图。"DIDN'T TOUCH" 部分尤为重要——它表明你遵循了范围纪律，没有进行未经请求的大规模改造。

## 提交前的检查

每次提交之前：

```bash
# 1. 检查即将提交的内容
git diff --staged

# 2. 确保没有密钥
git diff --staged | grep -i "password\|secret\|api_key\|token"

# 3. 运行测试
npm test

# 4. 运行 lint
npm run lint

# 5. 运行类型检查
npx tsc --noEmit
```

使用 git hooks 自动化这些步骤：

```json
// package.json（使用 lint-staged + husky）
{
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md}": ["prettier --write"]
  }
}
```

## 处理生成的文件

- **仅在项目需要时才提交生成的文件**（例如 `package-lock.json`、Prisma migration）
- **不要提交** 构建产物（`dist/`、`.next/`）、环境文件（`.env`）或 IDE 配置（`.vscode/settings.json`，除非是共享的）
- **使用 `.gitignore`** 覆盖以下内容：`node_modules/`、`dist/`、`.env`、`.env.local`、`*.pem`

## 使用 Git 进行调试

```bash
# 查找引入 bug 的 commit
git bisect start
git bisect bad HEAD
git bisect good <known-good-commit>
# Git 会检出中间点；在每次检出时运行测试以缩小范围

# 查看最近的变更
git log --oneline -20
git diff HEAD~5..HEAD -- src/

# 查找谁最后修改了某一行
git blame src/services/task.ts

# 在 commit 消息中搜索关键词
git log --grep="validation" --oneline
```

## 常见的自我合理化

| 合理化说辞 | 现实 |
|---|---|
| "等功能做完再提交" | 一次巨大的提交无法审查、调试或回退。每个切片都应该提交。 |
| "消息不重要" | 消息就是文档。未来的你（和未来的 agent）需要理解变更了什么以及为什么。 |
| "以后再把它们 squash" | squash 会破坏开发叙事。从一开始就保持干净的增量 commit。 |
| "分支增加了开销" | 短生命周期的分支没有成本，还能防止并行工作相互冲突。长期分支才是问题——在 1-3 天内合并。 |
| "以后再拆分这个变更" | 大型变更更难审查、部署风险更高、也更难回退。在提交前拆分，而不是之后。 |
| "我不需要 .gitignore" | 直到包含生产密钥的 `.env` 被提交为止。立即设置好。 |

## 危险信号

- 大量未提交的变更在积累
- commit message 类似 "fix"、"update"、"misc"
- 格式化变更与行为变更混合
- 项目中没有 `.gitignore`
- 提交了 `node_modules/`、`.env` 或构建产物
- 与 main 严重分歧的长期分支
- 向共享分支 force push

## 验证

对于每次 commit：

- [ ] commit 只做一件逻辑上的事情
- [ ] 消息解释了为什么，并遵循类型约定
- [ ] 提交前测试通过
- [ ] diff 中没有密钥
- [ ] 没有将仅格式化的变更与行为变更混合
- [ ] `.gitignore` 覆盖了标准的排除项
