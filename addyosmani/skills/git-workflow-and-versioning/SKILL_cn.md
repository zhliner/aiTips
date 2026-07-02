---
name: git-workflow-and-versioning
description: 结构化 git 工作流实践。做任何代码变更时使用。在提交、分支、解决冲突，或需要组织多个并行工作流时使用。
---

# Git 工作流与版本管理

## 概述

Git 是你的安全网。把提交视为存档点，把分支视为沙盒，把历史记录视为文档。在 AI agent 高速生成代码的情况下，有纪律的版本控制是让变更保持可管理、可审查和可回滚的机制。

## 何时使用

始终使用。每个代码变更都经过 git。

## 核心原则

### 基于主干的开发（推荐）

保持 `main` 始终可部署。在短生命周期的功能分支中工作，1-3 天内合并回主分支。长期存在的开发分支是隐性成本——它们会产生分歧、造成合并冲突并延迟集成。DORA 的研究一致表明，基于主干的开发与高绩效工程团队相关联。

```
main ──●──●──●──●──●──●──●──●──●──  (始终可部署)
        ╲      ╱  ╲    ╱
         ●──●─╱    ●──╱    ← 短生命周期功能分支 (1-3 天)
```

这是推荐的默认方式。使用 gitflow 或长期分支的团队可以将这些原则（原子提交、小变更、描述性消息）调整到他们的分支模型中——提交纪律比具体分支策略更重要。

- **开发分支是成本。** 分支每存在一天，就累积合并风险。
- **发布分支是可以接受的。** 当你需要在 main 前进的同时稳定一个版本时使用。
- **功能标志 > 长期分支。** 优先将未完成的工作部署在功能标志后面，而不是将其放在分支上数周。

### 1. 早提交，常提交

每个成功的增量都应有自己的提交。不要累积大量未提交的变更。

```
工作模式:
  实现一部分 → 测试 → 验证 → 提交 → 下一部分

不要这样:
  实现全部 → 希望它能正常工作 → 巨型提交
```

提交是存档点。如果下一个变更破坏了某些东西，你可以立即回滚到上一个已知良好的状态。

### 2. 原子提交

每个提交只做一件逻辑上完整的事情：

```
# 好：每个提交都是自包含的
git log --oneline
a1b2c3d 添加带验证的任务创建端点
d4e5f6g 添加任务创建表单组件
h7i8j9k 将表单连接到 API 并添加加载状态
m1n2o3p 添加任务创建测试（单元 + 集成）

# 坏：所有东西混在一起
git log --oneline
x1y2z3a 添加任务功能、修复侧边栏、更新依赖、重构工具函数
```

### 3. 描述性消息

提交消息解释*为什么*，而不仅仅是*是什么*：

```
# 好：解释意图
feat: 为注册端点添加邮箱验证

防止无效邮箱格式进入数据库。
在路由处理层使用 Zod schema 验证，
与 auth.ts 中现有的验证模式保持一致。

# 坏：描述 diff 中显而易见的内容
更新 auth.ts
```

**格式：**
```
<类型>: <简短描述>

<可选正文，解释为什么，而不是是什么>
```

**类型：**
- `feat` — 新功能
- `fix` — Bug 修复
- `refactor` — 既不修复 bug 也不添加功能的代码变更
- `test` — 添加或更新测试
- `docs` — 仅文档
- `chore` — 工具、依赖、配置

### 4. 关注点分离

不要把格式化变更和行为变更混在一起。不要把重构和功能混在一起。每种类型的变更都应该是独立的提交——理想情况下也应该是独立的 PR：

```
# 好：关注点分离
git commit -m "refactor: 将验证逻辑提取到共享工具函数"
git commit -m "feat: 为注册添加电话号码验证"

# 坏：混在一起
git commit -m "重构验证并添加电话号码字段"
```

**把重构和功能工作分开。** 重构变更和功能变更是两种不同的变更——分别提交。这样每个变更都更容易审查、回滚和理解历史记录。小的清理（重命名一个变量）可以视审查者意愿包含在功能提交中。

### 5. 控制变更大小

目标是每次提交/PR 约 100 行。超过约 1000 行的变更应当拆分。参见 `code-review-and-quality` 中关于如何拆分大型变更的策略。

```
~100 行   → 易于审查，易于回滚
~300 行   → 单个逻辑变更可以接受
~1000 行  → 拆分为更小的变更
```

## 分支策略

### 功能分支

```
main (始终可部署)
  │
  ├── feature/task-creation    ← 每个分支一个功能
  ├── feature/user-settings    ← 并行工作
  └── fix/duplicate-tasks      ← Bug 修复
```

- 从 `main`（或团队的默认分支）创建分支
- 保持分支短生命周期（1-3 天内合并）——长期分支是隐性成本
- 合并后删除分支
- 优先使用功能标志而非长期分支来管理未完成的功能

### 分支命名

```
feature/<简短描述>   → feature/task-creation
fix/<简短描述>       → fix/duplicate-tasks
chore/<简短描述>     → chore/update-deps
refactor/<简短描述>  → refactor/auth-module
```

## 使用 Worktrees

对于并行 AI agent 工作，使用 git worktrees 同时运行多个分支：

```bash
# 为功能分支创建 worktree
git worktree add ../project-feature-a feature/task-creation
git worktree add ../project-feature-b feature/user-settings

# 每个 worktree 是一个有自己分支的独立目录
# Agent 可以并行工作，互不干扰
ls ../
  project/              ← main 分支
  project-feature-a/    ← task-creation 分支
  project-feature-b/    ← user-settings 分支

# 完成后，合并并清理
git worktree remove ../project-feature-a
```

好处：
- 多个 agent 可以同时处理不同功能
- 无需切换分支（每个目录有自己的分支）
- 如果一个实验失败，删除 worktree 即可——不会有任何损失
- 变更是隔离的，直到显式合并

## 存档点模式

```
Agent 开始工作
    │
    ├── 做了一项变更
    │   ├── 测试通过？→ 提交 → 继续
    │   └── 测试失败？→ 回滚到上次提交 → 调查
    │
    ├── 做了另一项变更
    │   ├── 测试通过？→ 提交 → 继续
    │   └── 测试失败？→ 回滚到上次提交 → 调查
    │
    └── 功能完成 → 所有提交形成干净的历史记录
```

这种模式意味着你永远不会丢失超过一个增量的工作。如果 agent 失控，`git reset --hard HEAD` 可以将你带回上一个成功状态。

## 变更摘要

在任何修改之后，提供结构化的摘要。这使审查更容易，记录范围纪律，并突出意外变更：

```
CHANGES MADE:
- src/routes/tasks.ts: 为 POST 端点添加验证中间件
- src/lib/validation.ts: 使用 Zod 添加 TaskCreateSchema

THINGS I DIDN'T TOUCH (有意为之):
- src/routes/auth.ts: 有类似的验证缺口，但超出范围
- src/middleware/error.ts: 错误格式可以改进（单独任务）

POTENTIAL CONCERNS:
- Zod schema 是严格的——拒绝额外字段。确认这是否符合预期。
- 添加了 zod 作为依赖项（gzip 后 72KB）——已在 package.json 中
```

这种模式能及早发现错误的假设，并为审查者提供清晰的变更地图。"DIDN'T TOUCH" 部分尤其重要——它表明你遵守了范围纪律，没有进行未经请求的大翻修。

## 提交前检查

每次提交前：

```bash
# 1. 检查你即将提交的内容
git diff --staged

# 2. 确保没有 secrets
git diff --staged | grep -i "password\|secret\|api_key\|token"

# 3. 运行测试
npm test

# 4. 运行 lint
npm run lint

# 5. 运行类型检查
npx tsc --noEmit
```

使用 git hooks 自动化这些检查：

```json
// package.json (使用 lint-staged + husky)
{
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md}": ["prettier --write"]
  }
}
```

## 处理生成文件

- **提交生成文件**仅在项目需要时（如 `package-lock.json`、Prisma 迁移文件）
- **不要提交**构建输出（`dist/`、`.next/`）、环境文件（`.env`）或 IDE 配置（`.vscode/settings.json`，除非是共享的）
- **要有 `.gitignore`**，涵盖：`node_modules/`、`dist/`、`.env`、`.env.local`、`*.pem`

## 使用 Git 调试

```bash
# 找出哪个提交引入了 bug
git bisect start
git bisect bad HEAD
git bisect good <已知良好的提交>
# Git 会 checkout 到中间点；在每个点上运行你的测试以缩小范围

# 查看最近的变更
git log --oneline -20
git diff HEAD~5..HEAD -- src/

# 找出谁最后修改了某一行
git blame src/services/task.ts

# 在提交消息中搜索关键词
git log --grep="validation" --oneline
```

## 常见借口

| 借口 | 现实 |
|---|---|
| "功能做完了我再提交" | 一个巨型提交无法审查、调试或回滚。把每一部分都提交。 |
| "提交消息不重要" | 消息是文档。未来的你（和未来的 agent）需要理解变更了什么以及为什么。 |
| "我后面会全部 squash" | Squashing 会破坏开发叙事。从一开始就偏好干净的增量提交。 |
| "分支增加额外开销" | 短生命周期分支是免费的，可以防止冲突的工作碰撞。问题在于长期分支——在 1-3 天内合并。 |
| "我后面再拆分这个变更" | 大型变更更难审查，部署风险更高，更难回滚。在提交前拆分，而非提交后。 |
| "我不需要 .gitignore" | 直到包含生产 secrets 的 `.env` 被提交。立即设置。 |

## 警示信号

- 大量未提交的变更累积
- 像 "fix"、"update"、"misc" 这样的提交消息
- 格式化变更与行为变更混在一起
- 项目中没有 `.gitignore`
- 提交 `node_modules/`、`.env` 或构建产物
- 与 main 产生显著分歧的长期分支
- 对共享分支进行 force-push

## 验证

对于每个提交：

- [ ] 提交只做一件逻辑上完整的事情
- [ ] 消息解释为什么，遵循类型约定
- [ ] 提交前测试通过
- [ ] diff 中没有 secrets
- [ ] 没有格式化变更与行为变更混在一起
- [ ] `.gitignore` 涵盖标准排除项
