---
name: using-superpowers
description: 在开始任何对话时使用——建立如何查找和使用 skills 的规则，要求在任何响应（包括澄清问题）之前调用 skill
---

<SUBAGENT-STOP>
如果你被作为 subagent 派遣执行特定任务，忽略此 skill。
</SUBAGENT-STOP>

<EXTREMELY-IMPORTANT>
如果你认为某个 skill 有哪怕 1% 的可能性适用于你正在做的事情，你绝对必须调用该 skill。

如果某个 skill 适用于你的任务，你没有选择，必须使用它。

这不可协商。你无法为此找到借口。
</EXTREMELY-IMPORTANT>

## 规则

**在任何响应或行动之前调用相关或请求的 skills**——包括澄清问题、探索代码库或检查文件。如果最终发现不适用于当前情况，你可以不使用它。

**进入计划模式之前：** 如果还没有进行头脑风暴，先调用 brainstorming skill。

然后宣布 "Using [skill] to [purpose]" 并严格遵循该 skill。如果它有检查清单，为每一项创建一个 todo。

## Skill 优先级

当多个 skills 适用时，流程 skills 优先——它们设定方法，然后实现 skills（frontend-design 等）执行。Brainstorming 和 systematic-debugging 是 Superpowers 最常用的流程 skills，但规则适用于所有 skills。

- "Let's build X" → 先 superpowers:brainstorming，然后实现 skills。
- "Fix this bug" → 先 superpowers:systematic-debugging，然后领域 skills。

## 危险信号

以下想法意味着停止——你在找借口：

| 想法 | 现实 |
|------|------|
| "这只是一个简单问题" | 问题就是任务。检查 skills。 |
| "我需要更多上下文" | Skill 检查在澄清问题之前。 |
| "让我先探索代码库" | Skills 告诉你如何探索。先检查。 |
| "我可以快速检查 git/文件" | 文件缺少对话上下文。检查 skills。 |
| "让我先收集信息" | Skills 告诉你如何收集信息。 |
| "这不需要正式的 skill" | 如果 skill 存在，就使用它。 |
| "我记得这个 skill" | Skills 会演进。阅读当前版本。 |
| "这不算任务" | 行动 = 任务。检查 skills。 |
| "这个 skill 太大材小用了" | 简单的事情会变得复杂。使用它。 |
| "我先做这一件事" | 在做任何事之前检查。 |
| "这感觉很有成效" | 无纪律的行动浪费时间。Skills 防止这种情况。 |
| "我知道那是什么意思" | 知道概念 ≠ 使用 skill。调用它。 |

## 平台适配

如果你的 harness 出现在这里，阅读其参考文件以获取特殊说明：

- Codex：`references/codex-tools.md`
- Pi：`references/pi-tools.md`
- Antigravity：`references/antigravity-tools.md`

## 用户指令

用户指令（CLAUDE.md、AGENTS.md、GEMINI.md 等，直接请求）优先于 skills，skills 又优先于默认行为。只有当你的用户搭档明确要求时，才跳过 skill 工作流或指令。
