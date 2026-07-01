---
name: using-superpowers
description: 在开始任何对话时使用——建立如何查找和使用 skill 的规则，要求在做出任何响应（包括澄清性问题）之前必须调用 skill
---

<SUBAGENT-STOP>
如果你被分派为 subagent 来执行特定任务，请忽略此 skill。
</SUBAGENT-STOP>

<EXTREMELY-IMPORTANT>
如果你认为某个 skill 有哪怕 1% 的可能性适用于你正在做的事情，你**绝对必须**调用该 skill。

如果某个 skill 适用于你的任务，你别无选择。必须使用它。

这不可协商。你不能找任何借口逃避。
</EXTREMELY-IMPORTANT>

## 规则

**在任何响应或行动之前调用相关或请求的 skill**——包括澄清问题、探索代码库或检查文件。如果结果对当前情况不合适，可以不使用它。

**进入计划模式之前：** 如果你还没有进行头脑风暴，请先调用 brainstorming skill。

然后宣布"使用 [skill] 来 [目的]"，并严格遵循该 skill。如果它有检查清单，为每个条目创建一个 todo。

## Skill 优先级

当多个 skill 适用时，流程类 skill 优先——它们设定方法，然后实现类 skill（如 frontend-design 等）执行。brainstorming 和 systematic-debugging 是 Superpowers 最常用的流程类 skill，但此规则适用于所有 skill。

- "我们来构建 X" → 首先 superpowers:brainstorming，然后是实现类 skill。
- "修复这个 bug" → 首先 superpowers:systematic-debugging，然后是领域 skill。

## 危险信号

以下想法意味着**停止**——你在找借口：

| 想法 | 现实 |
|---------|---------|
| "这只是个简单问题" | 问题就是任务。检查 skill。 |
| "我需要更多上下文" | skill 检查在澄清问题**之前**。 |
| "让我先探索代码库" | skill 告诉你**如何**探索。先检查。 |
| "我可以快速检查 git/文件" | 文件缺少对话上下文。检查 skill。 |
| "让我先收集信息" | skill 告诉你**如何**收集信息。 |
| "这不需要正式的 skill" | 如果存在 skill，就使用它。 |
| "我记得这个 skill" | skill 会演变。阅读当前版本。 |
| "这不算是任务" | 行动 = 任务。检查 skill。 |
| "用 skill 太过了" | 简单的事会变复杂。使用它。 |
| "我先做这一件事" | 在做任何事**之前**先检查。 |
| "这感觉很高效" | 无纪律的行动浪费时间。skill 防止这个。 |
| "我知道那个是什么意思" | 知道概念 ≠ 使用 skill。调用它。 |

## 平台适配

如果你的 harness 出现在这里，阅读其参考文件以获取特殊说明：

- Codex: `references/codex-tools.md`
- Pi: `references/pi-tools.md`
- Antigravity: `references/antigravity-tools.md`

## 用户指令

用户指令（CLAUDE.md、AGENTS.md、GEMINI.md 等，以及直接请求）优先于 skill，而 skill 又覆盖默认行为。只有当你的伙伴明确告诉你时才跳过 skill 工作流或指令。
