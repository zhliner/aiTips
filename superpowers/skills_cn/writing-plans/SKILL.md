---
name: writing-plans
description: 当你有一份针对多步骤任务的规格说明（Spec）或需求时使用，并且应在开始修改代码之前使用。
---

# 编写 Plan

## 概述

编写全面的实现 plan，假设工程师对我们的代码库零上下文，且审美判断力有限。记录他们需要知道的一切：每个任务需要修改哪些文件、代码、测试、可能需要查阅的文档、如何测试。以小型任务的形式提供完整 plan。DRY。YAGNI。TDD。频繁 commit。

假设他们是熟练的开发者，但几乎不了解我们的工具集或问题领域。假设他们不太了解好的测试设计。

**开始时宣布：** "I'm using the writing-plans skill to create the implementation plan."

**上下文：** 如果在隔离的 worktree 中工作，它应该是在执行时通过 `using-git-worktrees` skill 创建的。

**Plan 保存位置：** `docs/plans/YYYY-MM-DD-<feature-name>.md`
- （用户对 plan 位置的偏好会覆盖此默认值）

## 范围检查

如果 spec 涵盖多个独立子系统，它应该在 brainstorming 阶段被拆分为子项目 spec。如果没有，建议拆分为独立的 plan——每个子系统一个。每个 plan 都应能独立产出可运行、可测试的软件。

## 文件结构

在定义任务之前，先规划哪些文件将被创建或修改，以及每个文件的职责。这是锁定分解决策的地方。

- 设计具有清晰边界和明确定义接口的单元。每个文件应有一个明确的职责。
- 你在可以一次性放入上下文的代码上推理效果最好，当文件专注时你的编辑更可靠。优先选择小而专注的文件，而非做太多事的大文件。
- 一起变更的文件应该放在一起。按职责拆分，而非按技术层。
- 在现有代码库中，遵循已有的模式。如果代码库使用大文件，不要单方面重组——但如果你正在修改的文件已变得难以管理，在 plan 中包含拆分是合理的。

此结构为任务分解提供依据。每个任务应产出独立的、有意义的自包含变更。

## 任务合理 sizing

任务是携带自身测试周期的最小单元，值得一次新 reviewer 的 gate。划定任务边界时：将设置、配置、脚手架和文档步骤折叠到需要它们的任务中；仅在 reviewer 可以有意义地拒绝一个任务而批准其相邻任务时才拆分。每个任务以一个可独立测试的交付物结束。

## 小型任务粒度

**每个步骤是一个操作（2-5 分钟）：**
- "编写失败的测试" - 步骤
- "运行它确保失败" - 步骤
- "实现使测试通过的最小代码" - 步骤
- "运行测试确保通过" - 步骤
- "Commit" - 步骤

## Plan 文档头

**每个 plan 必须以此头部开始：**

```markdown
# [Feature Name] 实现 Plan

> **给 agentic workers：** 必需子技能：使用 subagent-driven-development（推荐）或 executing-plans 逐任务实现此 plan。步骤使用复选框（`- [ ]`）语法进行跟踪。

**Goal:** [一句话描述构建什么]

**Architecture:** [2-3 句关于方法的描述]

**Tech Stack:** [关键技术/库]

## 全局约束

[spec 中项目级别的要求——版本下限、依赖限制、命名和文本规则、平台要求——每行一条，从 spec 中逐字复制精确值。每个任务的需求隐式包含此部分。]

---
```

## 任务结构

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

**Interfaces:**
- Consumes: [此任务从早期任务使用什么——精确签名]
- Produces: [后续任务依赖什么——精确函数名、参数和返回类型。任务的实现者只看到自己的任务；此块是他们了解相邻任务使用的名称和类型的方式。]

- [ ] **Step 1: 编写失败的测试**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: 运行测试验证失败**

Run: `pytest tests/path/test.py::test_name -v`
Expected: 失败，提示 "function not defined"

- [ ] **Step 3: 编写最小实现**

```python
def function(input):
    return expected
```

- [ ] **Step 4: 运行测试验证通过**

Run: `pytest tests/path/test.py::test_name -v`
Expected: 通过

- [ ] **Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## 禁止占位符

每个步骤必须包含工程师需要的实际内容。这些是 **plan 失败** ——绝不这样写：
- "TBD"、"TODO"、"稍后实现"、"补充细节"
- "添加适当的错误处理" / "添加验证" / "处理边界情况"
- "为上述内容编写测试"（不提供实际测试代码）
- "类似于 Task N"（重复代码——工程师可能不按顺序阅读任务）
- 步骤只描述做什么而不展示如何做（代码步骤需要代码块）
- 引用未在任何任务中定义的类型、函数或方法

## 记住
- 始终使用精确文件路径
- 每个步骤包含完整代码——如果步骤修改代码，展示代码
- 精确命令及预期输出
- DRY、YAGNI、TDD、频繁 commit

## 自审

编写完整 plan 后，以全新视角审视 spec，对照检查 plan。这是你自己运行的检查清单——不是 subagent 派遣。

**1. Spec 覆盖：** 浏览 spec 中的每个部分/需求。你能指向实现它的任务吗？列出任何差距。

**2. 占位符扫描：** 搜索 plan 中的危险信号——上述"禁止占位符"部分中的任何模式。修复它们。

**3. 类型一致性：** 你在后续任务中使用的类型、方法签名和属性名是否与早期任务中定义的一致？Task 3 中叫 `clearLayers()` 而 Task 7 中叫 `clearFullLayers()` 就是一个 bug。

如果发现问题，就地修复。无需重新 review——修复后继续。如果发现 spec 需求没有对应任务，添加任务。

## 执行交接

保存 plan 后，提供执行选择：

**"Plan 已完成并保存到 `docs/plans/<filename>.md`。两种执行选项：**

**1. Subagent-Driven（推荐）** - 我为每个任务派遣新的 subagent，任务间 review，快速迭代

**2. Inline Execution** - 在此 session 中使用 executing-plans 执行任务，批量执行并设置检查点

**选择哪种方式？"**

**如果选择 Subagent-Driven：**
- **REQUIRED SUB-SKILL：** 使用 subagent-driven-development
- 每个任务一个新 subagent + 两阶段 review

**如果选择 Inline Execution：**
- **REQUIRED SUB-SKILL：** 使用 executing-plans
- 批量执行并设置检查点供 review
