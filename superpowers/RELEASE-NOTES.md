# Superpowers Release Notes（发布说明）

## v4.3.1 (2026-02-21)

### Added（新增）

**Cursor 支持**

Superpowers 现在支持 Cursor 的插件系统。包含 `.cursor-plugin/plugin.json` 清单文件和 README 中 Cursor 特定的安装说明。SessionStart hook 输出现在包含 `additional_context` 字段，与现有的 `hookSpecificOutput.additionalContext` 并列，以兼容 Cursor hook。

### Fixed（修复）

**Windows：恢复多语言包装器以实现可靠的 hook 执行（#518、#504、#491、#487、#466、#440）**

Claude Code 在 Windows 上的 `.sh` 自动检测会在 hook 命令前添加 `bash`，破坏了执行。修复方案：

- 将 `session-start.sh` 重命名为 `session-start`（无扩展名），使自动检测不干扰
- 恢复 `run-hook.cmd` 多语言包装器，支持多位置 bash 发现（标准 Git for Windows 路径，然后 PATH 回退）
- 如果找不到 bash 则静默退出，不报错
- 在 Unix 上，包装器通过 `exec bash` 直接运行脚本
- 使用 POSIX 安全的 `dirname "$0"` 路径解析（在 dash/sh 上工作，不仅限于 bash）

这修复了 Windows 上 SessionStart 失败的问题，包括路径中有空格、缺少 WSL、MSYS 上 `set -euo pipefail` 脆弱性和反斜杠变形。

## v4.3.0 (2026-02-12)

此修复应显著改善 superpowers 技能合规性，并应减少 Claude 无意进入其原生计划模式的可能性。

### Changed（变更）

**Brainstorming 技能现在强制执行工作流而非描述它**

模型跳过设计阶段直接跳到实现技能（如 frontend-design），或将整个头脑风暴过程压缩为单个文本块。该技能现在使用硬门控、强制性清单和 graphviz 流程图来强制合规：

- `<HARD-GATE>`：在设计展示并获得用户批准之前，不使用实现技能、不写代码、不搭建框架
- 显式清单（6 项）必须创建为任务并按顺序完成
- Graphviz 流程图，`writing-plans` 是唯一有效的终止状态
- 反模式提示："这太简单了不需要设计"——正是模型用来跳过流程的合理化理由
- 设计章节大小基于章节复杂度，而非项目复杂度

**Using-superpowers 工作流图拦截 EnterPlanMode**

在技能流程图中添加了 `EnterPlanMode` 拦截。当模型即将进入 Claude 的原生计划模式时，它检查头脑风暴是否已发生，并路由通过头脑风暴技能。永远不会进入计划模式。

### Fixed（修复）

**SessionStart hook 现在同步运行**

在 hooks.json 中将 `async: true` 改为 `async: false`。异步时，hook 可能在模型第一轮之前未完成，导致 using-superpowers 指令不在第一条消息的上下文中。

## v4.2.0 (2026-02-05)

### Breaking Changes（破坏性变更）

**Codex：用原生技能发现替换引导 CLI**

`superpowers-codex` 引导 CLI、Windows `.cmd` 包装器和相关引导内容文件已被移除。Codex 现在通过 `~/.agents/skills/superpowers/` 符号链接使用原生技能发现，因此旧的 `use_skill`/`find_skills` CLI 工具不再需要。

安装现在只需 clone + 符号链接（在 INSTALL.md 中有文档）。不再需要 Node.js 依赖。旧的 `~/.codex/skills/` 路径已被弃用。

### Fixes（修复）

**Windows：修复 Claude Code 2.1.x hook 执行（#331）**

Claude Code 2.1.x 改变了 Windows 上 hook 的执行方式：它现在自动检测命令中的 `.sh` 文件并在前面添加 `bash`。这破坏了多语言包装器模式，因为 `bash "run-hook.cmd" session-start.sh` 会尝试将 `.cmd` 文件作为 bash 脚本执行。

修复：hooks.json 现在直接调用 session-start.sh。Claude Code 2.1.x 自动处理 bash 调用。还添加了 .gitattributes 以强制 shell 脚本使用 LF 换行符（修复 Windows checkout 时的 CRLF 问题）。

**Windows：SessionStart hook 异步运行以防止终端冻结（#404、#413、#414、#419）**

同步 SessionStart hook 阻止了 TUI 在 Windows 上进入原始模式，冻结了所有键盘输入。异步运行 hook 防止冻结，同时仍然注入 superpowers 上下文。

**Windows：修复 O(n^2) `escape_for_json` 性能**

使用 `${input:$i:1}` 的逐字符循环在 bash 中是 O(n^2)，因为子字符串复制开销。在 Windows Git Bash 上需要 60 多秒。替换为 bash 参数替换（`${s//old/new}`），每个模式作为单次 C 级别操作运行——macOS 上快 7 倍，Windows 上快得多。

**Codex：修复 Windows/PowerShell 调用（#285、#243）**

- Windows 不遵循 shebangs，因此直接调用无扩展名的 `superpowers-codex` 脚本会触发"打开方式"对话框。所有调用现在都以 `node` 为前缀。
- 修复 Windows 上的 `~/` 路径展开——PowerShell 在将 `~` 作为参数传递给 `node` 时不展开。改为 `$HOME`，在 bash 和 PowerShell 中都能正确展开。

**Codex：修复安装程序中的路径解析**

使用 `fileURLToPath()` 代替手动 URL pathname 解析，以正确处理所有平台上带空格和特殊字符的路径。

**Codex：修复 writing-skills 中过时的技能路径**

将 `~/.codex/skills/` 引用（已弃用）更新为 `~/.agents/skills/` 以用于原生发现。

### Improvements（改进）

**Worktree 隔离现在在实现前必需**

添加 `using-git-worktrees` 作为 `subagent-driven-development` 和 `executing-plans` 的必需技能。实现工作流现在明确要求在开始工作前设置隔离的 worktree，防止意外在 main 上直接工作。

**主分支保护放宽为需要明确同意**

技能不再完全禁止在 main 分支上工作，而是在用户明确同意下允许。更灵活，同时确保用户了解影响。

**简化安装验证**

从验证步骤中移除了 `/help` 命令检查和特定斜杠命令列表。技能主要通过描述你想做的事情来调用，而不是运行特定命令。

**Codex：在引导中澄清子代理工具映射**

改进了 Codex 工具如何映射到 Claude Code 等效工具（用于子代理工作流）的文档。

### Tests（测试）

- 为 subagent-driven-development 添加了 worktree 需求测试
- 添加了主分支红旗警告测试
- 修复了技能识别测试断言中的大小写敏感性

---

## v4.1.1 (2026-01-23)

### Fixes（修复）

**OpenCode：按官方文档标准化为 `plugins/` 目录（#343）**

OpenCode 官方文档使用 `~/.config/opencode/plugins/`（复数）。我们的文档之前使用 `plugin/`（单数）。虽然 OpenCode 接受两种形式，但我们已标准化为官方约定以避免混淆。

变更：
- 在仓库结构中将 `.opencode/plugin/` 重命名为 `.opencode/plugins/`
- 更新了所有平台的所有安装文档（INSTALL.md、README.opencode.md）
- 更新了测试脚本以匹配

**OpenCode：修复符号链接指令（#339、#342）**

- 在 `ln -s` 之前添加了显式 `rm`（修复重新安装时"文件已存在"错误）
- 添加了 INSTALL.md 中缺少的技能符号链接步骤
- 从已弃用的 `use_skill`/`find_skills` 更新为原生 `skill` 工具引用

---

## v4.1.0 (2026-01-23)

### Breaking Changes（破坏性变更）

**OpenCode：切换到原生技能系统**

OpenCode 的 Superpowers 现在使用 OpenCode 原生的 `skill` 工具，而不是自定义的 `use_skill`/`find_skills` 工具。这是更简洁的集成，与 OpenCode 的内置技能发现配合工作。

**需要迁移：** 技能必须符号链接到 `~/.config/opencode/skills/superpowers/`（参见更新的安装文档）。

### Fixes（修复）

**OpenCode：修复会话启动时的代理重置（#226）**

之前使用 `session.prompt({ noReply: true })` 的引导注入方法导致 OpenCode 在第一条消息时将选定代理重置为 "build"。现在使用 `experimental.chat.system.transform` hook，直接修改系统提示而无副作用。

**OpenCode：修复 Windows 安装（#232）**

- 移除了对 `skills-core.js` 的依赖（消除文件被复制而非符号链接时的损坏相对导入）
- 为 cmd.exe、PowerShell 和 Git Bash 添加了全面的 Windows 安装文档
- 记录了每个平台正确的符号链接与连接点用法

**Claude Code：修复 Claude Code 2.1.x 的 Windows hook 执行**

Claude Code 2.1.x 改变了 Windows 上 hook 的执行方式：它现在自动检测命令中的 `.sh` 文件并在前面添加 `bash `。这破坏了多语言包装器模式，因为 `bash "run-hook.cmd" session-start.sh` 会尝试将 .cmd 文件作为 bash 脚本执行。

修复：hooks.json 现在直接调用 session-start.sh。Claude Code 2.1.x 自动处理 bash 调用。还添加了 .gitattributes 以强制 shell 脚本使用 LF 换行符（修复 Windows checkout 时的 CRLF 问题）。

---

## v4.0.3 (2025-12-26)

### Improvements（改进）

**加强 using-superpowers 技能以处理显式技能请求**

解决了一种失败模式：即使用户按名称显式请求技能（例如 "subagent-driven-development, please"），Claude 也会跳过调用。Claude 会想"我知道那是什么意思"然后直接开始工作而不是加载技能。

变更：
- 更新了"The Rule"，说"调用相关或请求的技能"而不是"检查技能"——强调主动调用而非被动检查
- 添加了"在任何响应或行动之前"——原始措辞只提到"响应"，但 Claude 有时会在不响应的情况下先采取行动
- 添加了调用错误技能也没关系的保证——减少犹豫
- 添加了新的红旗信号："I know what that means" → 知道概念 ≠ 使用技能

**添加了显式技能请求测试**

在 `tests/explicit-skill-requests/` 中添加了新测试套件，验证 Claude 在用户按名称请求时正确调用技能。包含单轮和多轮测试场景。

## v4.0.2 (2025-12-23)

### Fixes（修复）

**斜杠命令现在仅限用户使用**

在所有三个斜杠命令（`/brainstorm`、`/execute-plan`、`/write-plan`）中添加了 `disable-model-invocation: true`。Claude 不再能通过 Skill 工具调用这些命令——它们仅限于手动用户调用。

底层技能（`superpowers:brainstorming`、`superpowers:executing-plans`、`superpowers:writing-plans`）仍然可供 Claude 自主调用。此变更防止了 Claude 调用一个只是重定向到技能的命令时的混淆。

## v4.0.1 (2025-12-23)

### Fixes（修复）

**澄清了如何在 Claude Code 中访问技能**

修复了一种令人困惑的模式：Claude 通过 Skill 工具调用技能，然后尝试单独 Read 技能文件。`using-superpowers` 技能现在明确说明 Skill 工具直接加载技能内容——不需要读文件。

- 在 `using-superpowers` 中添加了"How to Access Skills"章节
- 在指令中将"read the skill"改为"invoke the skill"
- 更新了斜杠命令以使用完全限定的技能名称（例如 `superpowers:brainstorming`）

**在 receiving-code-review 中添加了 GitHub 线程回复指导**（感谢 @ralphbean）

添加了关于在原始线程中回复内联审查评论而不是作为顶级 PR 评论的说明。

**在 writing-skills 中添加了自动化优于文档的指导**（感谢 @EthanJStark）

添加了指导：机械性约束应该自动化而非文档化——将技能留给判断性决策。

## v4.0.0 (2025-12-17)

### New Features（新功能）

**subagent-driven-development 中的两阶段代码审查**

子代理工作流现在在每个任务后使用两个独立的审查阶段：

1. **规格合规性审查** - 持怀疑态度的审查者验证实现是否完全匹配规格。捕获缺失需求和过度构建。不信任实现者的报告——阅读实际代码。

2. **代码质量审查** - 仅在规格合规性通过后运行。审查代码整洁度、测试覆盖率、可维护性。

这捕获了常见的失败模式：代码写得很好但不符合请求。审查是循环的，不是一次性的：如果审查者发现问题，实现者修复，然后审查者再次检查。

其他子代理工作流改进：
- 控制器向工作者提供完整任务文本（不是文件引用）
- 工作者可以在工作之前和期间提出澄清问题
- 报告完成前的自我审查清单
- 计划在开始时读取一次，提取到 TodoWrite

在 `skills/subagent-driven-development/` 中的新提示模板：
- `implementer-prompt.md` - 包含自我审查清单，鼓励提问
- `spec-reviewer-prompt.md` - 对照需求的怀疑验证
- `code-quality-reviewer-prompt.md` - 标准代码审查

**调试技术整合与工具**

`systematic-debugging` 现在捆绑了支持技术和工具：
- `root-cause-tracing.md` - 通过调用栈向后追踪 bug
- `defense-in-depth.md` - 在多个层添加验证
- `condition-based-waiting.md` - 用条件轮询替换任意超时
- `find-polluter.sh` - 查找哪个测试造成污染的二分脚本
- `condition-based-waiting-example.ts` - 来自真实调试会话的完整实现

**测试反模式参考**

`test-driven-development` 现在包含 `testing-anti-patterns.md`，涵盖：
- 测试 mock 行为而非真实行为
- 在生产类中添加仅测试方法
- 不理解依赖就 mock
- 隐藏结构假设的不完整 mock

**技能测试基础设施**

三个新的测试框架用于验证技能行为：

`tests/skill-triggering/` - 验证技能从朴素提示触发而无需显式命名。测试 6 个技能以确保仅描述就足够。

`tests/claude-code/` - 使用 `claude -p` 进行无头测试的集成测试。通过会话记录（JSONL）分析验证技能使用。包含用于成本跟踪的 `analyze-token-usage.py`。

`tests/subagent-driven-dev/` - 端到端工作流验证，包含两个完整测试项目：
- `go-fractals/` - 带 Sierpinski/Mandelbrot 的 CLI 工具（10 个任务）
- `svelte-todo/` - 带 localStorage 和 Playwright 的 CRUD 应用（12 个任务）

### Major Changes（重大变更）

**DOT 流程图作为可执行规范**

使用 DOT/GraphViz 流程图作为权威流程定义重写了关键技能。散文成为支持内容。

**The Description Trap**（记录在 `writing-skills` 中）：发现当描述包含工作流摘要时，技能描述会覆盖流程图内容。Claude 遵循简短描述而不是阅读详细流程图。修复：描述必须仅为触发型（"Use when X"），不包含流程细节。

**using-superpowers 中的技能优先级**

当多个技能适用时，流程技能（brainstorming、debugging）现在明确优先于实现技能。"Build X" 先触发 brainstorming，然后是领域技能。

**brainstorming 触发条件加强**

描述改为祈使句："You MUST use this before any creative work—creating features, building components, adding functionality, or modifying behavior."

### Breaking Changes（破坏性变更）

**技能整合** - 六个独立技能合并：
- `root-cause-tracing`、`defense-in-depth`、`condition-based-waiting` → 捆绑在 `systematic-debugging/` 中
- `testing-skills-with-subagents` → 捆绑在 `writing-skills/` 中
- `testing-anti-patterns` → 捆绑在 `test-driven-development/` 中
- `sharing-skills` 移除（已过时）

### Other Improvements（其他改进）

- **render-graphs.js** - 从技能中提取 DOT 图并渲染为 SVG 的工具
- **using-superpowers 中的合理化表格** - 可扫描格式，包括新条目："I need more context first"、"Let me explore first"、"This feels productive"
- **docs/testing.md** - 使用 Claude Code 集成测试测试技能的指南

---

## v3.6.2 (2025-12-03)

### Fixed（修复）

- **Linux 兼容性**：修复多语言 hook 包装器（`run-hook.cmd`）以使用 POSIX 兼容语法
  - 将 bash 特有的 `${BASH_SOURCE[0]:-$0}` 替换为标准 `$0`（第 16 行）
  - 解决 Ubuntu/Debian 系统上 `/bin/sh` 为 dash 时的 "Bad substitution" 错误
  - 修复 #141

---

## v3.5.1 (2025-11-24)

### Changed（变更）

- **OpenCode 引导重构**：从 `chat.message` hook 切换到 `session.created` 事件进行引导注入
  - 引导现在通过 `session.prompt()` 在会话创建时注入，带 `noReply: true`
  - 明确告诉模型 using-superpowers 已加载，防止冗余技能加载
  - 将引导内容生成整合到共享的 `getBootstrapContent()` 辅助函数
  - 更清晰的单一实现方法（移除回退模式）

---

## v3.5.0 (2025-11-23)

### Added（新增）

- **OpenCode 支持**：OpenCode.ai 的原生 JavaScript 插件
  - 自定义工具：`use_skill` 和 `find_skills`
  - 消息插入模式，在上下文压缩后保持技能
  - 通过 chat.message hook 自动上下文注入
  - 在 session.compacted 事件时自动重新注入
  - 三层技能优先级：项目 > 个人 > superpowers
  - 项目本地技能支持（`.opencode/skills/`）
  - 共享核心模块（`lib/skills-core.js`）用于与 Codex 的代码复用
  - 带适当隔离的自动化测试套件（`tests/opencode/`）
  - 平台特定文档（`docs/README.opencode.md`、`docs/README.codex.md`）

### Changed（变更）

- **重构 Codex 实现**：现在使用共享的 `lib/skills-core.js` ES 模块
  - 消除 Codex 和 OpenCode 之间的代码重复
  - 技能发现和解析的单一事实来源
  - Codex 通过 Node.js 互操作成功加载 ES 模块

- **改进文档**：重写 README 以清晰解释问题/解决方案
  - 移除重复章节和冲突信息
  - 添加完整工作流描述（头脑风暴 → 计划 → 执行 → 完成）
  - 简化平台安装说明
  - 强调技能检查协议而非自动激活声明

---

## v3.4.1 (2025-10-31)

### Improvements（改进）

- 优化 superpowers 引导以消除冗余技能执行。`using-superpowers` 技能内容现在直接在会话上下文中提供，并明确指导仅对其他技能使用 Skill 工具。这减少了开销，防止了代理在会话开始时已经拥有内容的情况下仍手动执行 `using-superpowers` 的混乱循环。

## v3.4.0 (2025-10-30)

### Improvements（改进）

- 简化 `brainstorming` 技能以回归最初的对话愿景。移除了带正式清单的繁重 6 阶段流程，转而使用自然对话：一次问一个问题，然后以 200-300 字的章节展示设计并验证。保留文档和实现交接功能。

## v3.3.1 (2025-10-28)

### Improvements（改进）

- 更新 `brainstorming` 技能，要求在提问前自主侦察，鼓励以建议驱动的决策，并防止代理将优先级排序委托回人类。
- 按照 Strunk 的"Elements of Style"原则对 `brainstorming` 技能应用了写作清晰度改进（省略不必要的词语，将否定形式转为肯定形式，改善平行结构）。

### Bug Fixes（Bug 修复）

- 澄清了 `writing-skills` 指导，使其指向正确的代理特定个人技能目录（Claude Code 为 `~/.claude/skills`，Codex 为 `~/.codex/skills`）。

## v3.3.0 (2025-10-28)

### New Features（新功能）

**实验性 Codex 支持**
- 添加了统一的 `superpowers-codex` 脚本，带 bootstrap/use-skill/find-skills 命令
- 跨平台 Node.js 实现（适用于 Windows、macOS、Linux）
- 命名空间技能：`superpowers:skill-name` 用于 superpowers 技能，`skill-name` 用于个人技能
- 名称匹配时个人技能覆盖 superpowers 技能
- 清晰的技能展示：显示名称/描述而非原始 frontmatter
- 有用的上下文：显示每个技能的支持文件目录
- Codex 的工具映射：TodoWrite→update_plan，子代理→手动回退等
- 带最小 AGENTS.md 的引导集成，用于自动启动
- 完整的安装指南和 Codex 特定的引导说明

**与 Claude Code 集成的关键差异：**
- 单一统一脚本而非独立工具
- Codex 特定等效工具的工具替换系统
- 简化的子代理处理（手动工作而非委托）
- 更新术语："Superpowers skills" 而非 "Core skills"

### Files Added（新增文件）
- `.codex/INSTALL.md` - Codex 用户的安装指南
- `.codex/superpowers-bootstrap.md` - 带 Codex 适配的引导说明
- `.codex/superpowers-codex` - 包含所有功能的统一 Node.js 可执行文件

**注意：** Codex 支持是实验性的。集成提供了核心 superpowers 功能，但可能需要根据用户反馈进行完善。

## v3.2.3 (2025-10-23)

### Improvements（改进）

**更新 using-superpowers 技能使用 Skill 工具代替 Read 工具**
- 将技能调用指令从 Read 工具改为 Skill 工具
- 更新描述："using Read tool" → "using Skill tool"
- 更新步骤 3："Use the Read tool" → "Use the Skill tool to read and run"
- 更新合理化列表："Read the current version" → "Run the current version"

Skill 工具是在 Claude Code 中调用技能的正确机制。此更新修正了引导指令，引导代理使用正确的工具。

### Files Changed（变更文件）
- 更新：`skills/using-superpowers/SKILL.md` - 将工具引用从 Read 改为 Skill

## v3.2.2 (2025-10-21)

### Improvements（改进）

**加强 using-superpowers 技能以抵御代理合理化**
- 添加了 EXTREMELY-IMPORTANT 块，使用绝对语言关于强制性技能检查
  - "If even 1% chance a skill applies, you MUST read it"
  - "You do not have a choice. You cannot rationalize your way out."
- 添加了 MANDATORY FIRST RESPONSE PROTOCOL 清单
  - 代理在任何响应之前必须完成的 5 步流程
  - 显式"不遵循此流程就响应 = 失败"后果
- 添加了 Common Rationalizations 章节，包含 8 种特定逃避模式
  - "This is just a simple question" → WRONG
  - "I can check files quickly" → WRONG
  - "Let me gather information first" → WRONG
  - 加上在代理行为中观察到的 5 种更常见模式

这些变更解决了观察到的代理行为，即尽管有明确指令，它们仍合理化跳过技能使用。强力的语言和预防性的反驳论点旨在使不合规变得更难。

### Files Changed（变更文件）
- 更新：`skills/using-superpowers/SKILL.md` - 添加了三层强制执行以防止跳过技能的合理化

## v3.2.1 (2025-10-20)

### New Features（新功能）

**代码审查者代理现在包含在插件中**
- 在插件的 `agents/` 目录中添加了 `superpowers:code-reviewer` 代理
- 代理提供根据计划和编码标准的系统代码审查
- 之前需要用户有个人代理配置
- 所有技能引用更新为使用命名空间的 `superpowers:code-reviewer`
- 修复 #55

### Files Changed（变更文件）
- 新增：`agents/code-reviewer.md` - 带审查清单和输出格式的代理定义
- 更新：`skills/requesting-code-review/SKILL.md` - 引用 `superpowers:code-reviewer`
- 更新：`skills/subagent-driven-development/SKILL.md` - 引用 `superpowers:code-reviewer`

## v3.2.0 (2025-10-18)

### New Features（新功能）

**brainstorming 工作流中的设计文档**
- 在 brainstorming 技能中添加了阶段 4：设计文档
- 设计文档现在在实现之前写入 `docs/plans/YYYY-MM-DD-<topic>-design.md`
- 恢复了原始 brainstorming 命令中在技能转换期间丢失的功能
- 文档在 worktree 设置和实现计划之前编写
- 使用子代理测试以验证时间压力下的合规性

### Breaking Changes（破坏性变更）

**技能引用命名空间标准化**
- 所有内部技能引用现在使用 `superpowers:` 命名空间前缀
- 更新格式：`superpowers:test-driven-development`（之前仅为 `test-driven-development`）
- 影响所有 REQUIRED SUB-SKILL、RECOMMENDED SUB-SKILL 和 REQUIRED BACKGROUND 引用
- 与使用 Skill 工具调用技能的方式对齐
- 更新的文件：brainstorming、executing-plans、subagent-driven-development、systematic-debugging、testing-skills-with-subagents、writing-plans、writing-skills

### Improvements（改进）

**设计与实现计划的命名**
- 设计文档使用 `-design.md` 后缀以防止文件名冲突
- 实现计划继续使用现有的 `YYYY-MM-DD-<feature-name>.md` 格式
- 两者都存储在 `docs/plans/` 目录中，命名区分清晰

## v3.1.1 (2025-10-17)

### Bug Fixes（Bug 修复）

- **修复 README 中的命令语法**（#44）- 更新所有命令引用以使用正确的命名空间语法（`/superpowers:brainstorm` 而非 `/brainstorm`）。插件提供的命令由 Claude Code 自动添加命名空间，以避免插件之间的冲突。

## v3.1.0 (2025-10-17)

### Breaking Changes（破坏性变更）

**技能名称标准化为小写**
- 所有技能 frontmatter 的 `name:` 字段现在使用小写 kebab-case，匹配目录名
- 示例：`brainstorming`、`test-driven-development`、`using-git-worktrees`
- 所有技能公告和交叉引用更新为小写格式
- 这确保了目录名、frontmatter 和文档之间的命名一致

### New Features（新功能）

**增强的 brainstorming 技能**
- 添加了快速参考表，显示阶段、活动和工具使用
- 添加了可复制的工作流清单以跟踪进度
- 添加了何时回顾早期阶段的决策流程图
- 添加了全面的 AskUserQuestion 工具指导和具体示例
- 添加了"Question Patterns"章节，解释何时使用结构化与开放式问题
- 将 Key Principles 重构为可扫描表格

**Anthropic 最佳实践集成**
- 添加了 `skills/writing-skills/anthropic-best-practices.md` - 官方 Anthropic 技能编写指南
- 在 writing-skills SKILL.md 中引用以提供全面指导
- 提供渐进式披露、工作流和评估的模式

### Improvements（改进）

**技能交叉引用清晰度**
- 所有技能引用现在使用显式需求标记：
  - `**REQUIRED BACKGROUND:**` - 你必须理解的先决条件
  - `**REQUIRED SUB-SKILL:**` - 工作流中必须使用的技能
  - `**Complementary skills:**` - 可选但有帮助的相关技能
- 移除了旧路径格式（`skills/collaboration/X` → 仅 `X`）
- 更新了 Integration 章节，带分类关系（Required vs Complementary）
- 更新了交叉引用文档与最佳实践

**与 Anthropic 最佳实践的对齐**
- 修复了描述语法和语态（完全第三人称）
- 添加了可扫描的快速参考表
- 添加了 Claude 可以复制和跟踪的工作流清单
- 对非显而易见的决策点适当使用流程图
- 改善了可扫描表格格式
- 所有技能都在 500 行建议以下

### Bug Fixes（Bug 修复）

- **重新添加丢失的命令重定向** - 恢复了在 v3.0 迁移中意外移除的 `commands/brainstorm.md` 和 `commands/write-plan.md`
- 修复了 `defense-in-depth` 名称不匹配（之前为 `Defense-in-Depth-Validation`）
- 修复了 `receiving-code-review` 名称不匹配（之前为 `Code-Review-Reception`）
- 修复了 `commands/brainstorm.md` 对正确技能名称的引用
- 移除了对不存在的相关技能的引用

### Documentation（文档）

**writing-skills 改进**
- 更新了带显式需求标记的交叉引用指导
- 添加了对 Anthropic 官方最佳实践的引用
- 改善了显示正确技能引用格式的示例

## v3.0.1 (2025-10-16)

### Changes（变更）

我们现在使用 Anthropic 的第一方技能系统！

## v2.0.2 (2025-10-12)

### Bug Fixes（Bug 修复）

- **修复本地技能仓库领先上游时的虚假警告** - 初始化脚本在本地仓库有领先上游的提交时错误地警告 "New skills available from upstream"。逻辑现在正确区分三种 git 状态：本地落后（应更新）、本地领先（无警告）和分叉（应警告）。

## v2.0.1 (2025-10-12)

### Bug Fixes（Bug 修复）

- **修复插件上下文中的 session-start hook 执行**（#8、PR #9）- hook 以 "Plugin hook error" 静默失败，阻止技能上下文加载。修复方案：
  - 当 BASH_SOURCE 在 Claude Code 执行上下文中未绑定时使用 `${BASH_SOURCE[0]:-$0}` 回退
  - 添加 `|| true` 以在过滤状态标志时优雅处理空 grep 结果

---

# Superpowers v2.0.0 Release Notes（发布说明）

## Overview（概述）

Superpowers v2.0 通过重大架构变革使技能更易访问、更可维护、更社区驱动。

标题变更是**技能仓库分离**：所有技能、脚本和文档已从插件移至专用仓库（[obra/superpowers-skills](https://github.com/obra/superpowers-skills)）。这将 superpowers 从单体插件转变为管理技能仓库本地克隆的轻量级垫片。技能在会话启动时自动更新。用户通过 fork 和标准 git 工作流贡献改进。技能库独立于插件进行版本管理。

除基础设施外，此版本添加了九个专注于问题解决、研究和架构的新技能。我们重写了核心 **using-skills** 文档，使用祈使语气和更清晰的结构，使 Claude 更容易理解何时以及如何使用技能。**find-skills** 现在输出可以直接粘贴到 Read 工具中的路径，消除了技能发现工作流中的摩擦。

用户体验无缝：插件自动处理克隆、fork 和更新。贡献者发现新架构使改进和分享技能变得微不足道。此版本为技能作为社区资源快速进化奠定了基础。

## Breaking Changes（破坏性变更）

### Skills Repository Separation（技能仓库分离）

**最大的变更：** 技能不再存在于插件中。它们已被移至独立仓库 [obra/superpowers-skills](https://github.com/obra/superpowers-skills)。

**对你的影响：**

- **首次安装：** 插件自动克隆技能到 `~/.config/superpowers/skills/`
- **Forking：** 在设置期间，你可以选择 fork 技能仓库（如果安装了 `gh`）
- **更新：** 技能在会话启动时自动更新（尽可能快进合并）
- **贡献：** 在分支上工作，本地提交，向上游提交 PR
- **不再有覆盖：** 旧的两层系统（个人/核心）被单仓库分支工作流取代

**迁移：**

如果你有现有安装：
1. 旧的 `~/.config/superpowers/.git` 将被备份到 `~/.config/superpowers/.git.bak`
2. 旧技能将被备份到 `~/.config/superpowers/skills.bak`
3. 在 `~/.config/superpowers/skills/` 创建 obra/superpowers-skills 的新克隆

### Removed Features（移除的功能）

- **个人 superpowers 覆盖系统** - 被 git 分支工作流取代
- **setup-personal-superpowers hook** - 被 initialize-skills.sh 取代

## New Features（新功能）

### Skills Repository Infrastructure（技能仓库基础设施）

**自动克隆和设置**（`lib/initialize-skills.sh`）
- 首次运行时克隆 obra/superpowers-skills
- 如果安装了 GitHub CLI 则提供 fork 创建
- 正确设置 upstream/origin 远程
- 处理从旧安装的迁移

**自动更新**
- 每次会话启动时从跟踪远程获取
- 尽可能自动快进合并
- 需要手动同步时通知（分支分叉）
- 使用 pulling-updates-from-skills-repository 技能进行手动同步

### New Skills（新技能）

**Problem-Solving Skills（问题解决技能）**（`skills/problem-solving/`）
- **collision-zone-thinking** - 强制不相关概念碰撞以产生涌现洞察
- **inversion-exercise** - 翻转假设以揭示隐藏约束
- **meta-pattern-recognition** - 发现跨领域的通用原则
- **scale-game** - 在极端情况下测试以暴露基本事实
- **simplification-cascades** - 发现能消除多个组件的洞察
- **when-stuck** - 分派到正确的问题解决技术

**Research Skills（研究技能）**（`skills/research/`）
- **tracing-knowledge-lineages** - 理解想法如何随时间演变

**Architecture Skills（架构技能）**（`skills/architecture/`）
- **preserving-productive-tensions** - 保持多个有效方法而非强制过早解决

### Skills Improvements（技能改进）

**using-skills（原名 getting-started）**
- 从 getting-started 重命名为 using-skills
- 使用祈使语气的完整重写（v4.0.0）
- 前置关键规则
- 为所有工作流添加了"Why"解释
- 引用中始终包含 /SKILL.md 后缀
- 更清晰地区分刚性规则和灵活模式

**writing-skills**
- 从 using-skills 移入交叉引用指导
- 添加了 token 效率章节（字数目标）
- 改善了 CSO（Claude Search Optimization）指导

**sharing-skills**
- 更新为新的分支和 PR 工作流（v2.0.0）
- 移除了个人/核心分割引用

**pulling-updates-from-skills-repository**（新增）
- 与上游同步的完整工作流
- 替换旧的 "updating-skills" 技能

### Tools Improvements（工具改进）

**find-skills**
- 现在输出带 /SKILL.md 后缀的完整路径
- 使路径可直接用于 Read 工具
- 更新了帮助文本

**skill-run**
- 从 scripts/ 移至 skills/using-skills/
- 改善了文档

### Plugin Infrastructure（插件基础设施）

**Session Start Hook**
- 现在从技能仓库位置加载
- 在会话启动时显示完整技能列表
- 打印技能位置信息
- 显示更新状态（成功更新 / 落后上游）
- 将"技能落后"警告移至输出末尾

**Environment Variables（环境变量）**
- `SUPERPOWERS_SKILLS_ROOT` 设置为 `~/.config/superpowers/skills`
- 在所有路径中一致使用

## Bug Fixes（Bug 修复）

- 修复 fork 时重复添加上游远程
- 修复 find-skills 输出中的双重 "skills/" 前缀
- 从 session-start 中移除过时的 setup-personal-superpowers 调用
- 修复 hooks 和命令中的路径引用

## Documentation（文档）

### README
- 更新为新的技能仓库架构
- 突出显示 superpowers-skills 仓库链接
- 更新自动更新描述
- 修复技能名称和引用
- 更新 Meta 技能列表

### Testing Documentation（测试文档）
- 添加了全面的测试清单（`docs/TESTING-CHECKLIST.md`）
- 创建了用于测试的本地市场配置
- 记录了手动测试场景

## Technical Details（技术细节）

### File Changes（文件变更）

**新增：**
- `lib/initialize-skills.sh` - 技能仓库初始化和自动更新
- `docs/TESTING-CHECKLIST.md` - 手动测试场景
- `.claude-plugin/marketplace.json` - 本地测试配置

**移除：**
- `skills/` 目录（82 个文件）- 现在在 obra/superpowers-skills 中
- `scripts/` 目录 - 现在在 obra/superpowers-skills/skills/using-skills/ 中
- `hooks/setup-personal-superpowers.sh` - 已过时

**修改：**
- `hooks/session-start.sh` - 使用 ~/.config/superpowers/skills 中的技能
- `commands/brainstorm.md` - 更新路径到 SUPERPOWERS_SKILLS_ROOT
- `commands/write-plan.md` - 更新路径到 SUPERPOWERS_SKILLS_ROOT
- `commands/execute-plan.md` - 更新路径到 SUPERPOWERS_SKILLS_ROOT
- `README.md` - 为新架构完整重写

### Commit History（提交历史）

此版本包含：
- 20+ 次技能仓库分离提交
- PR #1：受 Amplifier 启发的问题解决和研究技能
- PR #2：个人 superpowers 覆盖系统（后来被替换）
- 多次技能完善和文档改进

## Upgrade Instructions（升级说明）

### Fresh Install（全新安装）

```bash
# 在 Claude Code 中
/plugin marketplace add obra/superpowers-marketplace
/plugin install superpowers@superpowers-marketplace
```

插件自动处理一切。

### Upgrading from v1.x（从 v1.x 升级）

1. **备份你的个人技能**（如果有的话）：
   ```bash
   cp -r ~/.config/superpowers/skills ~/superpowers-skills-backup
   ```

2. **更新插件：**
   ```bash
   /plugin update superpowers
   ```

3. **下次会话启动时：**
   - 旧安装将自动备份
   - 新的技能仓库将被克隆
   - 如果你有 GitHub CLI，将提供 fork 选项

4. **迁移个人技能**（如果之前有的话）：
   - 在本地技能仓库中创建分支
   - 从备份复制个人技能
   - 提交并推送到你的 fork
   - 考虑通过 PR 回馈社区

## What's Next（后续计划）

### For Users（给用户）

- 探索新的问题解决技能
- 尝试基于分支的技能改进工作流
- 向社区贡献技能

### For Contributors（给贡献者）

- 技能仓库现在位于 https://github.com/obra/superpowers-skills
- Fork → Branch → PR 工作流
- 参见 skills/meta/writing-skills/SKILL.md 了解 TDD 文档方法

## Known Issues（已知问题）

目前没有已知问题。

## Credits（致谢）

- 问题解决技能受 Amplifier 模式启发
- 社区贡献和反馈
- 对技能有效性的大量测试和迭代

---

**完整变更日志：** https://github.com/obra/superpowers/compare/dd013f6...main
**技能仓库：** https://github.com/obra/superpowers-skills
**问题反馈：** https://github.com/obra/superpowers/issues
