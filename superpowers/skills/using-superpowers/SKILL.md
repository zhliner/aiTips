---
name: using-superpowers
description: 在开始任何对话时使用——建立如何查找和使用技能的规则，要求在任何响应（包括澄清性问题）之前调用 Skill 工具
---

<EXTREMELY-IMPORTANT>
如果你认为某个技能有哪怕 1% 的可能性适用于你正在做的事情，你绝对必须调用该技能。

如果某个技能适用于你的任务，你别无选择。你必须使用它。

这不可商量。这不是可选的。你不能找借口逃避。
</EXTREMELY-IMPORTANT>

## 如何访问技能

**在 Claude Code 中：** 使用 `Skill` 工具。当你调用一个技能时，其内容会被加载并展示给你——直接按照它执行。不要使用 Read 工具读取技能文件。

**在其他环境中：** 查阅你的平台文档了解技能加载方式。

# 使用技能（Using Skills）

## 规则

**在任何响应或行动之前调用相关或请求的技能。** 即使只有 1% 的可能性某个技能适用，你也应该调用它来检查。如果调用的技能不适用于当前情况，你不需要使用它。

```dot
digraph skill_flow {
    "User message received" [shape=doublecircle];
    "About to EnterPlanMode?" [shape=doublecircle];
    "Already brainstormed?" [shape=diamond];
    "Invoke brainstorming skill" [shape=box];
    "Might any skill apply?" [shape=diamond];
    "Invoke Skill tool" [shape=box];
    "Announce: 'Using [skill] to [purpose]'" [shape=box];
    "Has checklist?" [shape=diamond];
    "Create TodoWrite todo per item" [shape=box];
    "Follow skill exactly" [shape=box];
    "Respond (including clarifications)" [shape=doublecircle];

    "About to EnterPlanMode?" -> "Already brainstormed?";
    "Already brainstormed?" -> "Invoke brainstorming skill" [label="no"];
    "Already brainstormed?" -> "Might any skill apply?" [label="yes"];
    "Invoke brainstorming skill" -> "Might any skill apply?";

    "User message received" -> "Might any skill apply?";
    "Might any skill apply?" -> "Invoke Skill tool" [label="yes, even 1%"];
    "Might any skill apply?" -> "Respond (including clarifications)" [label="definitely not"];
    "Invoke Skill tool" -> "Announce: 'Using [skill] to [purpose]'";
    "Announce: 'Using [skill] to [purpose]'" -> "Has checklist?";
    "Has checklist?" -> "Create TodoWrite todo per item" [label="yes"];
    "Has checklist?" -> "Follow skill exactly" [label="no"];
    "Create TodoWrite todo per item" -> "Follow skill exactly";
}
```

## 危险信号

以下想法意味着停止——你在自我合理化：

| 想法 | 现实 |
|------|------|
| "这只是个简单问题" | 问题就是任务。检查技能。 |
| "我需要先获取更多上下文" | 技能检查在澄清性问题之前。 |
| "让我先探索代码库" | 技能告诉你如何探索。先检查。 |
| "我可以快速检查 git/文件" | 文件缺少对话上下文。检查技能。 |
| "让我先收集信息" | 技能告诉你如何收集信息。 |
| "这不需要正式的技能" | 如果技能存在，就使用它。 |
| "我记得这个技能" | 技能在演进。阅读当前版本。 |
| "这不算一个任务" | 行动 = 任务。检查技能。 |
| "这个技能小题大做了" | 简单的事情会变得复杂。使用它。 |
| "我先只做这一件事" | 在做任何事之前检查。 |
| "这感觉很有成效" | 无纪律的行动浪费时间。技能可以防止这种情况。 |
| "我知道那是什么意思" | 知道概念 ≠ 使用技能。调用它。 |

## 技能优先级

当多个技能可能适用时，按以下顺序使用：

1. **流程技能优先**（brainstorming、debugging）- 这些决定如何处理任务
2. **实现技能其次**（frontend-design、mcp-builder）- 这些指导执行

"Let's build X" → 先 brainstorming，然后实现技能。
"Fix this bug" → 先 debugging，然后领域特定技能。

## 技能类型

**严格型**（TDD、debugging）：严格遵循。不要削弱纪律。

**灵活型**（模式）：根据上下文调整原则。

技能本身会告诉你它属于哪种类型。

## 用户指令

指令说的是做什么（WHAT），而不是怎么做（HOW）。"Add X" 或 "Fix Y" 并不意味着跳过工作流。
