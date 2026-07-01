---
name: pre-creating
description: "在创造性工作之前使用此技能——创建功能、构建组件、添加功能或修改行为。在实现之前探索用户意图、需求和设计。"
---

# 将创意转化为设计（Valuable Ideas Into Designs）

## 概述

通过自然的协作对话，帮助将想法转化为完整的设计和规格说明。

首先了解当前项目的上下文，然后逐个提问来完善想法。一旦你理解了要构建什么，展示设计并获得用户批准。

<HARD-GATE>
在你展示设计并获得用户批准之前，不要调用任何实现技能、编写任何代码、搭建任何项目框架或采取任何实现行动。这适用于每个项目，无论其看起来多么简单。
</HARD-GATE>

## 反模式："这太简单了，不需要设计"

每个项目都要经过这个流程。一个 todo 列表、一个单函数工具、一个配置更改——全都要走这个流程。"简单"的项目正是未经审视的假设造成最多浪费的地方。设计可以很短（对真正简单的项目只需几句话），但你必须展示它并获得批准。

## 检查清单

你必须为以下每个项目创建任务并按顺序完成：

1. **探索项目上下文** — 检查文件、文档、最近的提交
2. **提出澄清性问题** — 逐个提问，理解目的/约束/成功标准
3. **提出 2-3 种方案** — 附带权衡和你的推荐
4. **展示设计** — 按复杂度分节展示，每节后获得用户批准
5. **编写设计文档** — 保存到 `docs/plans/YYYY-MM-DD-<topic>-design.md` 并提交
6. **过渡到实现** — 调用 writing-plans 技能创建实现计划

## 流程图

```dot
digraph pre-creating {
    "Explore project context" [shape=box];
    "Ask clarifying questions" [shape=box];
    "Propose 2-3 approaches" [shape=box];
    "Present design sections" [shape=box];
    "User approves design?" [shape=diamond];
    "Write design doc" [shape=box];
    "Invoke writing-plans skill" [shape=doublecircle];

    "Explore project context" -> "Ask clarifying questions";
    "Ask clarifying questions" -> "Propose 2-3 approaches";
    "Propose 2-3 approaches" -> "Present design sections";
    "Present design sections" -> "User approves design?";
    "User approves design?" -> "Present design sections" [label="no, revise"];
    "User approves design?" -> "Write design doc" [label="yes"];
    "Write design doc" -> "Invoke writing-plans skill";
}
```

**终止状态是调用 writing-plans。** 不要调用 frontend-design、mcp-builder 或任何其他实现技能。pre-creating 头脑风暴之后唯一调用的技能是 writing-plans。

## 流程

**理解想法：**
- 首先查看当前项目状态（文件、文档、最近的提交）
- 逐个提问来完善想法
- 尽可能使用选择题，但开放式问题也可以
- 每条消息只问一个问题——如果某个话题需要更多探索，将其拆分为多个问题
- 重点理解：目的、约束、成功标准

**探索方案：**
- 提出 2-3 种不同方案及其权衡
- 以对话方式展示选项，附上你的推荐和理由
- 先展示推荐方案并解释原因

**展示设计：**
- 一旦你认为理解了要构建什么，展示设计
- 每个部分的篇幅与其复杂度匹配：简单明了的部分用几句话，细微差别的部分可用 200-300 字
- 每部分之后询问到目前为止是否正确
- 涵盖：架构、组件、数据流、错误处理、测试
- 准备好在有不明白的地方时回头澄清

## 设计完成之后

**文档：**
- 将验证过的设计写入 `docs/plans/YYYY-MM-DD-<topic>-design.md`
- 如果可用，使用 elements-of-style:writing-clearly-and-concisely 技能
- 将设计文档提交到 git

**实现：**
- 调用 writing-plans 技能创建详细的实现计划
- 不要调用任何其他技能。writing-plans 是下一步。

## 关键原则

- **每次一个问题** - 不要用多个问题淹没用户
- **优先使用选择题** - 在可能的情况下比开放式问题更容易回答
- **严格遵循 YAGNI** - 从所有设计中移除不必要的功能
- **探索替代方案** - 在确定之前始终提出 2-3 种方案
- **增量验证** - 展示设计，获得批准后再继续
- **保持灵活** - 当某些东西不合理时回头澄清
