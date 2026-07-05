> **注意：** 本仓库包含 Anthropic 为 Claude 实现的 Skills。有关 Agent Skills 标准的信息，请参阅 [agentskills.io](http://agentskills.io)。

[![skills.sh](https://skills.sh/b/anthropics/skills)](https://skills.sh/anthropics/skills)

# Skills（技能）
Skills 是由说明、脚本和资源组成的文件夹，Claude 会动态加载它们以提升在专业任务上的表现。Skills 以可重复的方式教会 Claude 如何完成特定任务，无论是按照公司品牌指南创建文档、使用组织特定的工作流分析数据，还是自动化个人任务。

更多信息，请查看：
- [什么是 Skills？](https://support.claude.com/en/articles/12512176-what-are-skills)
- [在 Claude 中使用 Skills](https://support.claude.com/en/articles/12512180-using-skills-in-claude)
- [如何创建自定义 Skills](https://support.claude.com/en/articles/12512198-creating-custom-skills)
- [用 Agent Skills 为智能体赋能真实世界](https://anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)

# 关于本仓库

本仓库包含的 Skills 展示了 Claude Skills 系统的能力。这些 Skills 涵盖创意应用（艺术、音乐、设计）、技术任务（测试 Web 应用、生成 MCP server）到企业工作流（沟通、品牌等）。

每个 Skill 都包含在独立的文件夹中，其中有一个 `SKILL.md` 文件，包含 Claude 使用的说明和元数据。浏览这些 Skills 可以为创建自己的 Skills 获取灵感，或了解不同的模式和方法。

本仓库中的许多 Skills 是开源的（Apache 2.0）。我们还在 [`skills/docx`](./skills/docx)、[`skills/pdf`](./skills/pdf)、[`skills/pptx`](./skills/pptx) 和 [`skills/xlsx`](./skills/xlsx) 子文件夹中提供了驱动 [Claude 文档能力](https://www.anthropic.com/news/create-files)的文档创建与编辑 Skills。这些是源码可见（source-available）的，而非开源的，但我们希望与开发者分享这些内容，作为在生产级 AI 应用中积极使用的复杂 Skills 的参考。

## 免责声明

**这些 Skills 仅供演示和教育目的。** 虽然其中一些功能可能在 Claude 中可用，但你从 Claude 获得的实现和行为可能与这些 Skills 中展示的有所不同。这些 Skills 旨在展示模式和可能性。在将 Skills 用于关键任务之前，请务必在你自己的环境中进行充分测试。

# Skill 集合
- [./skills](./skills)：创意与设计、开发与技术、企业与沟通以及文档 Skills 的示例
- [./spec](./spec)：Agent Skills 规范
- [./template](./template)：Skill 模板

# 在 Claude Code、Claude.ai 和 API 中试用

## Claude Code
你可以在 Claude Code 中运行以下命令，将本仓库注册为 Claude Code Plugin 市场：
```
/plugin marketplace add anthropics/skills
```

然后，安装特定的 Skill 集合：
1. 选择 `Browse and install plugins`
2. 选择 `anthropic-agent-skills`
3. 选择 `document-skills` 或 `example-skills`
4. 选择 `Install now`

或者，直接通过以下命令安装 Plugin：
```
/plugin install document-skills@anthropic-agent-skills
/plugin install example-skills@anthropic-agent-skills
```

安装 Plugin 后，只需提及即可使用 Skill。例如，如果你从市场安装了 `document-skills` Plugin，你可以让 Claude Code 执行类似以下操作："使用 PDF Skill 从 `path/to/some-file.pdf` 中提取表单字段"

## Claude.ai

这些示例 Skills 已在 Claude.ai 中向付费用户开放。

要使用本仓库中的任何 Skill 或上传自定义 Skills，请按照 [在 Claude 中使用 Skills](https://support.claude.com/en/articles/12512180-using-skills-in-claude#h_a4222fa77b) 中的说明操作。

## Claude API

你可以通过 Claude API 使用 Anthropic 预构建的 Skills，并上传自定义 Skills。更多信息请参阅 [Skills API 快速入门](https://docs.claude.com/en/api/skills-guide#creating-a-skill)。

# 创建基础 Skill

创建 Skills 非常简单——只需一个包含 YAML frontmatter 和说明的 `SKILL.md` 文件的文件夹。你可以使用本仓库中的 **template-skill** 作为起点：

```markdown
---
name: my-skill-name
description: 对该 Skill 的功能和使用场景的清晰描述
---

# My Skill Name

[在此添加 Claude 在该 Skill 激活时将遵循的说明]

## 示例
- 示例用法 1
- 示例用法 2

## 指南
- 指南 1
- 指南 2
```

Frontmatter 只需要两个字段：
- `name` - Skill 的唯一标识符（小写，用连字符代替空格）
- `description` - Skill 功能和使用时机的完整描述

下方的 Markdown 内容包含 Claude 将遵循的说明、示例和指南。更多详情请参阅 [如何创建自定义 Skills](https://support.claude.com/en/articles/12512198-creating-custom-skills)。

# 合作伙伴 Skills

Skills 是教会 Claude 更好地使用特定软件的绝佳方式。当我们看到合作伙伴提供的优秀示例 Skills 时，可能会在此处重点介绍其中一些：

- **Notion** - [Notion Skills for Claude](https://www.notion.so/notiondevs/Notion-Skills-for-Claude-28da4445d27180c7af1df7d8615723d0)
