---
name: writing-plans
description: 当你有一个 spec 或需求，需要处理多步骤任务时，在动手写代码之前使用
---

# 编写计划（Writing Plans）

## 概述

编写全面的实施计划，假设工程师对我们的代码库零背景且品味存疑。记录他们需要知道的一切：每个任务需要改哪些文件、代码、测试、可能需要查阅的文档、如何测试。以 bite-sized 粒度的任务给出完整计划。遵循 DRY、YAGNI、TDD 原则，频繁提交。

假设他们是有经验的开发者，但对我们的工具集和问题领域几乎一无所知。假设他们不太了解良好的测试设计。

**开始时声明：** "我正在使用 writing-plans skill 来创建实施计划。"

**上下文：** 如果在隔离的 worktree 中工作时，应在执行阶段通过 `superpowers:using-git-worktrees` skill 创建 worktree。

**计划保存到：** `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- （用户对计划位置的偏好设置会覆盖此默认值）

## 范围检查

如果 spec 覆盖了多个独立的子系统，它应该在 brainstorming 阶段就被拆分为子项目 spec。如果没有，建议将其拆分为独立的计划——每个子系统一个。每个计划应能独立产出可工作、可测试的软件。

## 文件结构

在定义任务之前，先规划出将要创建或修改的文件以及每个文件的职责。这是分解决策最终确定的地方。

- 设计具有清晰边界和明确定义接口的单元。每个文件应有且仅有一个清晰的职责。
- 你对能一次性装入上下文的代码推理效果最佳，而且当文件聚焦时编辑也更可靠。优先选择小而精的文件，避免文件过大、承担过多职责。
- 一起变化的文件应该放在一起。按职责而非按技术层拆分。
- 在既有代码库中，遵循已建立的模式。如果代码库使用大文件，不要单方面重构——但如果正在修改的文件已经变得臃肿，在计划中加入拆分是合理的。

这个结构为任务分解提供依据。每个任务应产出具有独立意义、自包含的变更。

## 任务合理粒度

任务是最小的单元，它有自己的测试周期，且值得由一位 reviewer 独立审核。在划定任务边界时：将 setup、配置、脚手架、文档步骤合并到需要它们的那个任务的交付物中；仅在 reviewer 可以合理拒绝一个任务而批准相邻任务时才进行拆分。每个任务以可独立测试的交付物结束。

## Bite-Sized 任务粒度

**每个步骤是一个动作（2-5 分钟）：**
- "编写失败的测试"——一个步骤
- "运行测试以确保其失败"——一个步骤
- "实现最小代码使测试通过"——一个步骤
- "运行测试并确保通过"——一个步骤
- "提交"——一个步骤

## 计划文档头部

**每个计划必须以以下头部开头：**

```markdown
# [功能名称] 实施计划

> **面向 agentic worker：** 必选 SUB-SKILL：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 来逐任务实施此计划。步骤使用 checkbox（`- [ ]`）语法进行跟踪。

**目标：** [一句话描述此计划构建什么]

**架构：** [2-3 句话描述技术方案]

**技术栈：** [关键技术/库]

## 全局约束

[spec 中项目级的要求——版本下限、依赖限制、命名和文案规则、平台要求——每行一个，精确值从 spec 逐字复制。每个任务的要求隐式包含此部分。]

---
```

## 任务结构

````markdown
### 任务 N：[组件名称]

**文件：**
- 创建：`exact/path/to/file.py`
- 修改：`exact/path/to/existing.py:123-145`
- 测试：`tests/exact/path/to/test.py`

**接口：**
- 消费： [此任务使用的来自之前任务的内容——精确的函数签名]
- 产出： [后续任务依赖的内容——精确的函数名、参数和返回值类型。任务的实现者只会看到自己的任务；此代码块是它们了解邻近任务名称和类型的方式。]

- [ ] **步骤 1：编写失败的测试**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **步骤 2：运行测试以验证其失败**

运行：`pytest tests/path/test.py::test_name -v`
预期：FAIL，提示 "function not defined"

- [ ] **步骤 3：编写最小实现**

```python
def function(input):
    return expected
```

- [ ] **步骤 4：运行测试以验证其通过**

运行：`pytest tests/path/test.py::test_name -v`
预期：PASS

- [ ] **步骤 5：提交**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## 禁止占位符

每个步骤必须包含工程师所需的实际内容。以下属于**计划失败**——永远不要写这些：
- "TBD"、"TODO"、"待实现"、"待填写细节"
- "添加适当的错误处理" / "添加验证" / "处理边界情况"
- "为上述内容编写测试"（但没有实际测试代码）
- "类似于任务 N"（应重复代码——工程师可能按非顺序阅读任务）
- 描述要做到什么但没有展示怎么做的步骤（代码步骤必须有代码块）
- 引用任何任务中未定义的类型、函数或方法

## 牢记
- 始终使用精确的文件路径
- 每个步骤都要有完整代码——如果步骤涉及修改代码，展示代码
- 精确的命令及预期输出
- DRY、YAGNI、TDD、频繁提交

## 自审

写完完整计划后，以全新视角审视 spec 并对照检查计划。这是一个自己运行的 checklist——不需要调度子代理。

**1. Spec 覆盖：** 浏览 spec 中的每个章节/需求。能否指出是哪个任务实现了它？列出任何遗漏。

**2. 占位符扫描：** 在计划中搜索红旗标记——以上"禁止占位符"章节中的任何模式。修正它们。

**3. 类型一致性：** 后续任务中使用的类型、方法签名和属性名称是否与之前任务中定义的匹配？函数在任务 3 中叫 `clearLayers()` 但在任务 7 中叫 `clearFullLayers()` 就是一个 bug。

如果发现问题，直接在原地修改。无需重新审查——修正后继续。如果发现 spec 中有需求没有对应的任务，添加任务。

## 执行交接

保存计划后，提供执行选项：

**"计划已完成，保存到 `docs/superpowers/plans/<filename>.md`。两种执行选项：**

**1. Subagent-Driven（推荐）** - 每个任务调度一个全新的 subagent，任务之间进行 review，快速迭代

**2. Inline Execution** - 在当前 session 中使用 executing-plans 执行任务，checkpoint 批次执行

**选哪个方案？"**

**如果选择 Subagent-Driven：**
- **必选 SUB-SKILL：** 使用 superpowers:subagent-driven-development
- 每个任务一个全新 subagent + 两阶段 review

**如果选择 Inline Execution：**
- **必选 SUB-SKILL：** 使用 superpowers:executing-plans
- 批次执行，设置 checkpoint 用于 review
