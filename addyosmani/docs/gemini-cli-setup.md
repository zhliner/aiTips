# 在 Gemini CLI 中使用 agent-skills

## 安装配置（Setup）

### 方式一：作为技能安装（推荐）

Gemini CLI 拥有原生技能系统，可自动发现 `.gemini/skills/` 或 `.agents/skills/` 目录中的 `SKILL.md` 文件。每个技能在匹配到你的任务时按需激活。

**从仓库安装：**

```bash
gemini skills install https://github.com/addyosmani/agent-skills.git --path skills
```

**或从本地克隆安装：**

```bash
git clone https://github.com/addyosmani/agent-skills.git
gemini skills install /path/to/agent-skills/skills/
```

**仅安装到特定工作区：**

```bash
gemini skills install /path/to/agent-skills/skills/ --scope workspace
```

工作区级别的技能安装到 `.gemini/skills/`（或 `.agents/skills/`）。用户级技能安装到 `~/.gemini/skills/`。

安装完成后，验证：

```
/skills list
```

Gemini CLI 会自动将技能名称和描述注入提示词。当识别到匹配的任务时，会在加载完整指令前请求你授权激活该技能。

### 方式二：GEMINI.md（持久上下文）

对于你想始终作为持久项目上下文加载（而非按需激活）的技能，将它们添加到项目的 `GEMINI.md`：

```bash
# 创建 GEMINI.md，将核心技能作为持久上下文
cat /path/to/agent-skills/skills/incremental-implementation/SKILL.md > GEMINI.md
echo -e "\n---\n" >> GEMINI.md
cat /path/to/agent-skills/skills/code-review-and-quality/SKILL.md >> GEMINI.md
```

你也可以通过从独立文件导入来模块化：

```markdown
# 项目指令

@skills/test-driven-development/SKILL.md
@skills/incremental-implementation/SKILL.md
```

使用 `/memory show` 验证已加载的上下文，使用 `/memory reload` 在修改后刷新。

> **技能 vs GEMINI.md：** 技能是按需激活的专业知识，仅在相关时加载，保持上下文窗口整洁。GEMINI.md 提供在每个提示词中都会加载的持久上下文。对阶段特定工作流使用技能，对始终生效的项目规范使用 GEMINI.md。

## 推荐配置（Recommended Configuration）

### 始终生效（GEMINI.md）（Always-On）

将这些作为每个会话的持久上下文添加：

- `incremental-implementation` — 以小型可验证的切片构建
- `code-review-and-quality` — 五维度审查

### 按需加载（技能）（On-Demand）

将这些安装为技能，仅在相关时激活：

- `test-driven-development` — 实现逻辑或修复 bug 时激活
- `spec-driven-development` — 开始新项目或新功能时激活
- `frontend-ui-engineering` — 构建 UI 时激活
- `security-and-hardening` — 安全审查时激活
- `performance-optimization` — 性能优化时激活

## 高级配置（Advanced Configuration）

### MCP 集成（MCP Integration）

此技能包中的许多技能利用 [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) 工具与环境交互。例如：

- `browser-testing-with-devtools` 使用 `chrome-devtools` MCP 扩展。
- `performance-optimization` 可以受益于性能相关的 MCP 工具。

要启用这些，请确保在 Gemini CLI 配置（`~/.gemini/config.json`）中安装了相关 MCP 扩展。

### 会话钩子（Session Hooks）

Gemini CLI 支持会话生命周期钩子。你可以使用这些钩子在会话开始时自动注入上下文或运行验证脚本。

要复制其他工具中的 `agent-skills` 体验，你可以配置 `SessionStart` 钩子来提醒可用技能或加载元技能。

### 显式上下文加载（Explicit Context Loading）

你可以在提示词中使用 `@` 符号显式加载任何技能到当前会话：

```markdown
使用 @skills/test-driven-development/SKILL.md 技能来实现这个修复。
```

当你想确保遵循特定工作流而不等待自动发现时，这很有用。

## 斜杠命令（Slash Commands）

该仓库在 `.gemini/commands/` 下提供了 8 个斜杠命令：7 个生命周期命令加上 `/webperf` 专项审计。从项目根目录运行时，Gemini CLI 会自动发现它们。

| 命令 | 功能 |
|------|------|
| `/spec` | 在编写代码前撰写结构化规格说明 |
| `/planning` | 将工作拆分为小型、可验证的任务 |
| `/build` | 增量实现下一个任务 |
| `/test` | 运行 TDD 工作流 — 红灯、绿灯、重构 |
| `/review` | 五维度代码审查 |
| `/code-simplify` | 在不改变行为的前提下降低复杂度 |
| `/ship` | 通过并行角色扇出执行上线前检查 |
| `/webperf` | 审计面向浏览器的应用的 Core Web Vitals 和性能问题 |

每个命令自动调用对应的技能 — 无需手动加载技能。

> **注意：** 请使用 `/planning` 而非 `/plan` — `/plan` 与 Gemini CLI 内部命令名冲突。

## 使用技巧（Usage Tips）

1. **优先使用技能而非 GEMINI.md** — 技能按需激活，保持上下文窗口聚焦。只有当你希望技能始终加载时才放入 GEMINI.md。
2. **技能描述很重要** — 每个 SKILL.md 的 frontmatter 中都有 `description` 字段，告诉智能体何时激活它。本仓库中的描述经过优化，可在所有支持的工具（Claude Code、Gemini CLI 等）中实现自动发现，清晰地说明技能*做什么*以及*何时*应被触发。
3. **使用智能体进行审查** — 在请求结构化代码审查时复制 `agents/code-reviewer.md` 的内容。
4. **结合参考资料** — 在处理测试或性能等特定质量领域时，参考 `references/` 中的清单。
