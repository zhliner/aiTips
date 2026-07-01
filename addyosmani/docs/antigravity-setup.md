# 在 Antigravity CLI (agy) 中使用 agent-skills

`agent-skills` 包可以作为原生插件安装到 Antigravity CLI（`agy`）中，使智能体能够访问结构化工作流、角色和自定义斜杠命令。

## 安装配置（Setup）

### 方式一：原生插件安装（推荐）

Antigravity CLI 拥有一流的插件系统，可注册技能、智能体和自定义命令。

**从远程仓库安装：**

```bash
agy plugin install https://github.com/addyosmani/agent-skills.git
```

**从本地克隆安装：**

1. 克隆仓库：
   ```bash
   git clone https://github.com/addyosmani/agent-skills.git
   ```
2. 使用 `agy` 安装插件：
   ```bash
   agy plugin install /path/to/agent-skills
   ```

这将验证插件并将其安装到全局 Antigravity 配置目录（`~/.gemini/antigravity-cli/plugins/agent-skills/`）。

### 方式二：从 Gemini CLI 导入

如果你已经在旧版 Gemini CLI 安装中安装了 `agent-skills`，可以直接导入：
```bash
agy plugin import gemini
```

安装完成后，验证已激活的插件：
```bash
agy plugin list
```

---

## 斜杠命令（Slash Commands）

该插件注册了 8 个自定义斜杠命令：7 个生命周期命令加上 `/webperf` 专项审计：

| 命令 | 功能 | 激活的技能 |
|------|------|-----------|
| `/spec` | 在编写代码前撰写结构化规格说明 | `spec-driven-development` |
| `/planning` | 将工作拆分为小型、可验证的任务 | `planning-and-task-breakdown` |
| `/build` | 增量实现下一个任务 | `incremental-implementation` |
| `/test` | 运行 TDD 工作流 — 红灯、绿灯、重构 | `test-driven-development` |
| `/review` | 五维度代码审查 | `code-review-and-quality` |
| `/code-simplify` | 在不改变行为的前提下降低复杂度 | `code-simplification` |
| `/ship` | 通过并行角色扇出执行上线前检查 | `shipping-and-launch` |
| `/webperf` | 审计面向浏览器的应用的 Core Web Vitals 和性能问题 | `web-performance-auditor` |

每个命令会自动调用对应的技能，并逐步引导智能体执行。

> **注意：** 请使用 `/planning` 而非 `/plan`，以避免与 Antigravity 内部的计划生成命令冲突。

---

## 技能与发现（Skills & Discovery）

Antigravity 会自动发现插件 `skills/` 目录中的技能。
* Antigravity 按需将用户任务和意图匹配到相关技能。
* 如果任务匹配某个技能，智能体会加载该技能并在执行前提示你授权。

---

## 验证与校验（Verification & Validation）

要验证本地插件结构是否正确且包含所有技能，运行：
```bash
agy plugin validate /path/to/agent-skills
```

---

## 工作原理（How It Works）

### 1. 按需技能激活（On-Demand Skill Activation）
Antigravity CLI 自动发现已安装插件 `skills/` 目录中的 `SKILL.md` 文件。利用每个技能 frontmatter 中的触发描述，智能体在检测到匹配的开发者意图时动态激活相应工作流。

例如，当你要求智能体：
- **设计一个新系统** &rarr; 它会建议/激活 `spec-driven-development`。
- **实现一个功能** &rarr; 它会激活 `incremental-implementation` 和 `test-driven-development`。
- **修复一个 bug** &rarr; 它会激活 `debugging-and-error-recovery`。

### 2. 专业智能体角色（Specialized Agent Personas）
该插件从 `agents/` 目录注册可复用的子智能体定义：
- `code-reviewer.md`
- `security-auditor.md`
- `test-engineer.md`

你可以在会话中直接调用这些角色，或在使用子智能体委派任务时使用。

---

## 配置与自定义（Configuration & Customization）

### 项目级强制执行（`AGENTS.md`）
要强制技能合规（例如要求在写代码前必须有规格说明或计划），请将 `AGENTS.md` 复制或链接到工作区根目录。Antigravity CLI 会读取此文件以使智能体的行为和规划阶段与你团队的规范保持一致。

### 沙盒模式（Sandbox Mode）
如果你想以受限的终端权限运行技能或脚本（在运行第三方验证测试时更安全），使用以下命令启动 CLI：

```bash
agy --sandbox
```

---

## 使用技巧（Usage Tips）

1. **保持插件更新：** 你可以使用以下命令更新 CLI 或检查更新的插件版本：
   ```bash
   agy update
   ```
2. **执行前审查：** 当智能体使用这些技能执行复杂的重构任务时，使用 `Ctrl+r` 进入**产物审查（Artifact Review）**界面，在代码提交前审查、编辑或批准代码。
3. **控制权限：** `--dangerously-skip-permissions` 标志仅在你信任的本地项目中使用，用于跳过手动工具审批提示。
