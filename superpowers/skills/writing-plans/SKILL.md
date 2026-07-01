---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
---

# Writing Plans（编写计划）

## 概述

编写全面的实现计划，假设工程师对我们的代码库零上下文且品味存疑。记录他们需要知道的一切：每个任务要涉及哪些文件、代码、测试、可能需要查阅的文档、如何测试。以小块任务的形式提供完整计划。DRY。YAGNI。TDD。频繁提交。

假设他们是熟练的开发者，但对我们的工具集或问题领域几乎一无所知。假设他们不太了解好的测试设计。

**在开始时宣布：** "I'm using the writing-plans skill to create the implementation plan."

**上下文：** 这应该在专用的 worktree 中运行（由 brainstorming 技能创建）。

**计划保存至：** `docs/plans/YYYY-MM-DD-<feature-name>.md`

## 小块任务粒度

**每个步骤是一个操作（2-5 分钟）：**
- "Write the failing test" - 一步
- "Run it to make sure it fails" - 一步
- "Implement the minimal code to make the test pass" - 一步
- "Run the tests and make sure they pass" - 一步
- "Commit" - 一步

## 计划文档头部

**每个计划必须以此头部开始：**

```markdown
# [Feature Name] Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

## 任务结构

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

**Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

**Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

**Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## 记住
- 始终使用精确的文件路径
- 计划中包含完整代码（不是"add validation"）
- 精确的命令和预期输出
- 使用 @ 语法引用相关技能
- DRY、YAGNI、TDD、频繁提交

## 执行交接

保存计划后，提供执行选择：

**"Plan complete and saved to `docs/plans/<filename>.md`. Two execution options:**

**1. Subagent-Driven (this session)** - I dispatch fresh subagent per task, review between tasks, fast iteration

**2. Parallel Session (separate)** - Open new session with executing-plans, batch execution with checkpoints

**Which approach?"**

**如果选择 Subagent-Driven：**
- **必需子技能：** Use superpowers:subagent-driven-development
- 留在当前会话
- 每个任务一个全新子代理 + 代码评审

**如果选择 Parallel Session：**
- 引导他们在 worktree 中打开新会话
- **必需子技能：** 新会话使用 superpowers:executing-plans