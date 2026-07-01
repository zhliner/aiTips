# 技能结构详解（Skill Anatomy）

本文档描述 agent-skills 技能文件的结构和格式。在贡献新技能或理解现有技能时，请将此作为指南。

## 文件位置（File Location）

每个技能位于 `skills/` 下的独立目录：

```
skills/
  skill-name/
    SKILL.md           # 必需：技能定义
    scripts/           # 可选：技能工作流使用的可运行辅助脚本
    supporting-file.md # 可选：按需加载的参考资料
```

`SKILL.md` 是唯一必需的文件。仅在技能确实需要可运行的辅助脚本时才添加 `scripts/`，对于纯 Markdown 技能则完全省略该目录。

## SKILL.md 格式（SKILL.md Format）

### Frontmatter（必需）（Frontmatter - Required）

```yaml
---
name: skill-name-with-hyphens
description: Guides agents through [task/workflow]. Use when [specific trigger conditions].
---
```

**规则：**
- `name`：小写，连字符分隔。必须与目录名匹配。
- `description`：以第三人称描述技能的功能，然后包含一个或多个明确的"Use when"触发条件。同时包含*做什么*和*何时做*。最多 1024 个字符。

**为什么重要：** 智能体通过阅读描述来发现技能。描述会被注入系统提示词，因此必须告诉智能体技能提供什么以及何时激活它。不要总结工作流 — 如果描述包含流程步骤，智能体可能会遵循摘要而非阅读完整技能。

### 标准章节（推荐模式）（Standard Sections - Recommended Pattern）

上述 frontmatter 契约是必需的。下面的章节布局是推荐模式，不是刚性模板：当等效标题能更清晰地服务同一目的时，可以使用。

```markdown
# 技能标题

## 概述（Overview）
一两句话解释这个技能做什么以及为什么重要。

## 何时使用（When to Use）
- 触发条件列表（症状、任务类型）
- 何时不使用（排除条件）

## [核心流程 / 工作流 / 步骤]
主要工作流，拆分为编号步骤或阶段。
在有帮助的地方包含代码示例。
在决策点使用流程图（ASCII）。

## [具体技术 / 模式]
针对特定场景的详细指导。
代码示例、模板、配置。

## 常见合理化借口（Common Rationalizations）
| 借口 | 现实 |
|------|------|
| 智能体用来跳过步骤的借口 | 为什么这个借口是错的 |

## 红旗（Red Flags）
- 表明技能被违反的行为模式
- 审查期间需要注意的事项

## 验证（Verification）
完成技能流程后，确认：
- [ ] 退出条件清单
- [ ] 证据要求
```

## 章节用途（Section Purposes）

### 概述（Overview）
技能的"电梯演讲"。应回答：这个技能做什么，为什么智能体应该遵循它？

### 何时使用（When to Use）
帮助智能体和人类判断此技能是否适用于当前任务。同时包含正向触发条件（"当 X 时使用"）和反向排除条件（"不适用于 Y"）。

### 核心流程（Core Process）
技能的核心。这是智能体遵循的逐步工作流。必须具体且可操作 — 不是模糊的建议。

**好的：** "运行 `npm test` 并验证所有测试通过"
**差的：** "确保测试能工作"

### 常见合理化借口（Common Rationalizations）
精心制作的技能中最独特的功能。这些是智能体用来跳过重要步骤的借口，配有驳斥。它们防止智能体合理化地逃避遵循流程。

想想每次智能体说过"我以后再添加测试"或"这足够简单，可以跳过规格说明" — 这些就放在这里，配上事实性的反驳。

### 红旗（Red Flags）
技能被违反的可观察迹象。在代码审查和自我监控期间很有用。

### 验证（Verification）
退出条件。智能体用来确认技能流程已完成的清单。每个勾选项都应该可以通过证据验证（测试输出、构建结果、截图等）。

## 辅助文件（Supporting Files）

仅在以下情况创建辅助文件：
- 参考资料超过 100 行（保持主 SKILL.md 聚焦）
- 需要代码工具或脚本
- 清单足够长，需要独立文件

50 行以下的模式和原则保持内联。

如果技能不需要可运行的辅助脚本，不要仅为了模仿其他技能而创建空的 `scripts/` 目录。空目录增加噪音但不改变技能的工作方式。

## 上下文效率（Context Efficiency）

技能按需加载：启动时只有技能名称和描述在上下文中。完整的 `SKILL.md` 仅在智能体判断技能相关时才加载。为了保持加载轻量：

- **保持 `SKILL.md` 在 500 行以内。** 将详细参考资料移到辅助文件中。
- **编写精确的描述。** 精确的描述帮助智能体在正确时机激活技能，其他时候跳过。
- **使用渐进式披露。** 引用仅在到达工作流对应阶段时才读取的辅助文件。
- **优先使用脚本而非内联代码。** 执行脚本不消耗上下文；只有其输出会。内联代码块在每次加载时都要付出代价。
- **保持文件引用一层深。** 从 `SKILL.md` 直接链接到辅助文件，而非通过中间文档链式引用。

## 脚本要求（Script Requirements）

当技能在 `scripts/` 下提供可运行的辅助脚本时，每个脚本遵循以下约定：

- 使用 `#!/bin/bash` shebang。
- 使用 `set -e` 实现快速失败行为。
- 状态消息写入 stderr：`echo "Message" >&2`。
- 机器可读输出（JSON）写入 stdout。
- 包含临时文件的清理 trap。
- 脚本路径引用为 `skills/<skill-name>/scripts/<script>.sh`（相对于仓库根目录）。

## 编写原则（Writing Principles）

1. **流程重于知识。** 技能是工作流，不是参考文档。是步骤，不是事实。
2. **具体重于笼统。** "运行 `npm test`" 优于 "验证测试"。
3. **证据重于假设。** 每个验证勾选项需要证据。
4. **反合理化。** 每个可能被跳过的步骤都需要在合理化表格中有反驳。
5. **渐进式披露。** 主 SKILL.md 是入口。辅助文件仅在需要时加载。
6. **Token 意识。** 每个章节必须证明其存在的合理性。如果删除它不会改变智能体行为，就删除它。

## 命名约定（Naming Conventions）

- 技能目录：`lowercase-hyphen-separated`（小写连字符分隔）
- 技能文件：`SKILL.md`（始终大写）
- 辅助文件：`lowercase-hyphen-separated.md`（小写连字符分隔）
- 参考资料：存储在项目根目录的 `references/` 中，不在技能目录内

## 跨技能引用（Cross-Skill References）

按名称引用其他技能：

```markdown
遵循 `test-driven-development` 技能编写测试。
如果构建失败，使用 `debugging-and-error-recovery` 技能。
```

不要在技能之间复制内容 — 引用并链接。

## 必需 vs 推荐（Required vs Recommended）

必需：

- 一个 `skills/<skill-name>/SKILL.md` 文件
- 有效的 YAML frontmatter，包含 `name` 和 `description`
- 描述同时包含技能的功能和使用时机

推荐：

- 上述标准章节流程
- 当它们对技能来说更自然时，可使用等效标题如 `How It Works`、`Core Process` 或 `Workflow`
- 仅在辅助文件有助于保持主 `SKILL.md` 聚焦时才创建
