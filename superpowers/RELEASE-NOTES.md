# Superpowers 发布说明

## v6.1.0 (2026-06-30)

### 降低每会话 Token 成本

`using-superpowers` bootstrap 会注入到每个会话中，因此其体积大小是持续的成本。本次发布对其及其指向的每个 harness 参考文件进行了精简，同时保留了行为塑造内容。

- **压缩了 `using-superpowers` bootstrap。** 将 graphviz 技能流程图替换为其表达的文本内容，将独立的 Instruction-Priority 部分合并到 User Instructions，删除了每个平台的"How to Access Skills"指南，并将 Platform Adaptation 指针精简为仍提供参考文件的 harness。完整的 Red Flags 理性化表格和用户指令优先级规则保持不变。
- **精简了每个 harness 的工具映射参考。** 冗长的动作到工具映射表重述了现代智能体已经遵循的指导。每个参考文件只保留了仍有意义的 harness 特定说明——子智能体分派、任务跟踪、指令文件路径——而 `claude-code-tools.md` 和 `copilot-tools.md`（已无 harness 特定内容）已被删除。

### Codex

- **Codex 可从市场安装。** Codex 市场要求市场根目录下有 `.agents/plugins/marketplace.json`；仓库只提供了 Claude 市场文件，因此 Codex 可以识别市场但找不到可安装的插件条目。仓库本地的 Codex 市场清单现在指向相同的仓库根目录，因此插件可从 Codex 安装。
- **Codex 不再附带 SessionStart hook。** Codex 能自行可靠地触发技能，而 bootstrap hook 恶化而非改善了用户体验。Codex hook 配置（`hooks-codex.json`）及其清单注册已被移除。

### Harness 支持

- **移除了 Gemini CLI 支持。** Google 于 2026-06-18 终止了 Gemini CLI；该扩展无法再安装或更新。Gemini 已从安装文档、支持子智能体的平台列表和评估 harness 描述中移除，其工具映射参考文件也已删除。

## v6.0.3 (2026-06-18)

### 子智能体驱动开发

- **SDD 临时文件移出 `.git/`。** Claude Code 将 `.git/` 视为受保护路径并拒绝智能体在其中写入，因此将报告写入 `.git/sdd/` 的实施者子智能体在执行中途会被阻止。任务简报、实施者报告、审查 diff 和进度账本现在位于工作树中一个自动忽略的 `.superpowers/sdd/` 目录中——不会出现在 `git status` 中，也不会被提交，并通过共享的 `sdd-workspace` 辅助工具按 worktree 解析。需要注意一点：因为该工作空间是 gitignore 的工作树临时文件，`git clean -fdx` 会删除进度账本；如果发生这种情况，可通过 `git log` 恢复。(#1780)

## v6.0.2 (2026-06-16)

### 安装修复

- **不再附带 `evals` 子模块。** 这对部分用户造成了插件安装失败，因此评估 harness 现在位于自己的独立仓库中，与发布的插件分离。(#1778, #1774)

## v6.0.1 (2026-06-16)

### Codex 修复

- **brainstorm 伴侣中的版本显示**——打包的 Codex 插件不带根 `package.json`，因此可视化伴侣报告其版本为"unknown"。`readSuperpowersVersion()` 现在在 `package.json` 不存在时回退到 `.codex-plugin/plugin.json`。
- **更干净的 Codex 插件同步**——sync-to-codex 脚本现在排除 `.gitmodules` 和 `.pre-commit-config.yaml`，防止仓库元数据进入打包的 Codex 插件。

## v6.0.0 (2026-06-16)

Superpowers 6.0 是一个大版本。核心亮点是重写了 `subagent-driven-development` 审查每个任务的方式——更便宜、更严格、更难被绕过。

虽然这些数字不会在每个 harness 和每个工作负载上都成立，但在我们的评估中，Claude Code 和 Codex 产生类似高质量结果的速度大约快了两倍，同时 token 消耗减少了近 50%。

此外还新增了三个 harness（Kimi Code、Pi 和 Antigravity），为 brainstorming 可视化伴侣提供了更好的安全模型，并重写了多个技能中的工具调用使其更加厂商中立。

### 可见变化

- **每个任务的两个审查者提示变为一个。** `spec-reviewer-prompt.md` 和 `code-quality-reviewer-prompt.md` 已被移除，替换为单个 `task-reviewer-prompt.md`。如果你直接分派旧文件，请切换到新文件。
- **旧的全局 worktree 目录已移除。** `using-git-worktrees` 和 `finishing-a-development-branch` 不再使用 `~/.config/superpowers/worktrees/`。Worktree 现在创建在项目中——如果已有 `.worktrees/` 或 `worktrees/` 则使用现有目录，否则创建新的 `.worktrees/`——除非你另有说明。

### 新 Harness 支持

Superpowers 现在可以在另外三个 harness 上运行。每个都附带自己的 bootstrap、工具映射参考和测试，每个在 README 中都有自己的安装部分。

- **Kimi Code**——插件清单、安装文档和清单测试；可从 Kimi 的市场或直接从仓库安装。（初始清单由 @qer 提供）
- **Pi**——一个会话启动扩展，注册技能并注入 `using-superpowers` bootstrap。Pi 具有原生技能支持，因此不需要兼容性 shim。
- **Antigravity (`agy`)**——直接安装插件并从第一条消息进行 bootstrap；已通过标准的"make a react todo list"验收测试进行端到端验证。

### 子智能体驱动开发

在真实项目上进行的一系列成本和质量实验重塑了控制器审查每个任务的方式。旧流程为每个任务运行两个审查者，并依赖控制器对模型选择和严重程度的判断——两者都被证明是昂贵的且容易绕过。新流程为每个任务运行一个审查者，以文件而非粘贴文本的方式交接工作，并从控制器手中收回了多个判断权。

- **每个任务一个审查者，两个裁决。** 单个 `task-reviewer-prompt.md` 一次性读取任务的 diff，并返回规范合规性裁决和质量裁决，因此一次修复即可清除两者。新增的"can't verify from the diff"裁决标记出那些依赖未被触及代码中的需求，由控制器自行检查。(#1538, #1543)
- **末尾一次总审查。** 运行结束时，使用最强模型进行一次全分支审查，而不是逐任务重新审查所有内容。
- **计划获得预检读取。** 在第一个任务之前，控制器检查计划中的内部冲突——以及计划中要求的任何审查者会标记为缺陷的内容——并一次性全部提出，而不是在执行中途跌跌撞撞地发现。
- **Diff 和任务文本以文件方式传递。** 粘贴的 diff 会永久停留在最昂贵的上下文中，而没有 diff 的审查者会手动重建——这是最大的审查者成本。两个新脚本 `task-brief` 和 `review-package` 将任务文本和审查 diff 写入文件供子智能体读取。
- **每次分派都声明模型。** 在没有指引的情况下，控制器完全停止指定模型——而未指定模型会静默地继承会话中最昂贵的模型，因此有一次运行中所有 26 个审查者都使用了最顶级模型。模板现在要求指定模型，并附有在工作允许时使用较便宜层级的指导。
- **控制器不能告诉审查者忽略什么。** 真实运行中发现控制器指导审查者跳过某个发现或将其标记为"at most Minor"，缺陷就这样溜走了。现明确禁止压制发现和预先评级严重程度，即使是计划本身要求的缺陷也会上报供你决定，而不是直接放行。
- **审查者是只读的且对解释持怀疑态度。** 审查不再触及工作树或分支——曾有一个审查者运行 `git checkout` 导致后续提交变成孤立的——实施者的"我故意不抽象这个"不再能说服审查者放过真正的发现。
- **更强的证据和报告。** 审查者对每个答案都附带文件和行号，实施者的报告移至文件并包含适用 TDD 时的 red/green 证据，进度账本使失去上下文的控制器能够恢复而不是重做已完成的工作。(#994)

### 编写计划

计划现在携带了控制器和审查者过去在每次分派时都需要重新推导的结构。

- **Global Constraints 块**列出了约束每个任务的规则——版本下限、依赖限制、命名和文案、精确值——逐字复制，以便它们真正到达下游的实施者和审查者。
- **每个任务的 Interfaces 块**明确命名每个任务消费和产出的内容，使只看到自己任务的实施者仍然知道其邻居的契约。
- **大小适中指导**将任务保持在值得进行自己的测试周期和审查者通过的规模，将设置、配置和文档折叠到需要它们的任务中。在测试中，以这种方式编写的计划只需要一轮修复，而对照组需要两到四轮——而且对照组还发布了一个真正的 bug。

### Brainstorming 可视化伴侣

可视化伴侣是一个小型的 Web 服务器，智能体在对话旁边打开它。它完全没有认证，因此在共享或远程机器上，任何能访问该端口的人都可以读取你的 brainstorm——或注入智能体视为你输入的事件。本次发布为其赋予了真正的安全模型，并使其在重启和连接中断后仍能存活。

- **每个会话一个密钥现在保护一切。** 智能体的 URL 携带一个一次性密钥，浏览器将其存储在 tab 作用域的 cookie 中，每个请求和 WebSocket 连接都必须出示该密钥。这关闭了无意中打开的本地标签页和可路由远程主机的门，包括源允许列表无法捕获的 DNS 重绑定情况。（Closes #1014）
- **文件服务器停留在其沙箱内。** 它拒绝符号链接、dotfiles 和任何试图爬出内容目录的路径，忽略 macOS 资源 fork 文件，并发送常用的 no-store 和拒绝框架化头。保存会话密钥的文件以仅限所有者模式写入。
- **仅在有用时才提供伴侣。** 技能在第一次出现"展示比文字更好"的问题时，作为独立消息提出使用伴侣，并尊重拒绝。接受后会在浏览器中打开第一个页面。（Closes #755）
- **在重启和不稳定连接中存活。** 给定项目目录后，服务器在重启之间保持相同的端口和密钥，因此打开的标签页可以简单地重新连接。页面自行重新连接，显示实时状态点，并在服务器离线时显示"paused"覆盖层。
- **更长的空闲寿命，更安全的关闭。** 空闲超时从 30 分钟增加到 4 小时，`stop-server.sh` 现在在发信号前确认自己拥有正确的进程，这样在重启后绝不会杀死不相关的 `node` 进程。(#1703)
- **Windows 启动加固**——统一了 shell 检测，Windows 现在依赖空闲超时来关闭，因为 Node 无法跨 MSYS2 跟踪 POSIX 进程所有权。

### 现有 Harness 更新

- **Codex** 现在通过自己的 SessionStart hook 而不是共享接线进行 bootstrap，Codex App 增加了安装部分和更完整的工具文档（web search、`AGENTS.md`、个人技能）。(#1540)
- **OpenCode** 在其插件、安装文档和 README 中获得了基于动作的工具映射，以及 bootstrap 缓存测试。
- **Cursor** 的清单中删除了 `agents` 和 `commands` 条目，因为这些目录已不再存在。

### 一套技能，所有 Harness

技能过去使用 Claude Code 的方言——"use the Task tool"、"put it in CLAUDE.md"。本次发布以你所做事情的术语重写了这些词汇（"dispatch a subagent"、"your instructions file"），并添加了每个 harness 的参考文件，将每个动作映射到正确的工具，并针对每个运行时进行了检查。将说"Claude"的文本改为"your agent"。

- **每个 harness 一个工具参考**位于 `skills/using-superpowers/references/`，涵盖 Claude Code、Codex、Copilot、Gemini、Pi 和 Antigravity。
- **`finishing-a-development-branch` 改为 forge 中立**——不再硬编码 `gh pr create`，因此智能体使用其所拥有的任何 forge 工具进行推送。(#1609)
- **一项重命名：** "Claude Search Optimization" 现在为 "Skill Discovery Optimization"，因为该技术不是 Claude 特有的。

### 编写技能

为技能作者增加了两项内容。

- **Match the Form to the Failure**——一个简短的表格，用于选择正确的指导形式。平面的"don't do X"对纪律性失误有效，但当问题是输出的**形态**时则适得其反，此时给出工作示例效果更好。该表格以及理性化部分更紧凑的范围引导作者选择真正有帮助的形式。
- **Micro-Test Wording**——在承诺使用某个措辞之前，一种廉价的检查方式：以无指导对照组为参考，少量采样几次，手动阅读每个结果，将运行间差异视为警告信号。

### 测试

技能行为测试从 `tests/` 移出，进入一个新的 `evals/` 子模块，该模块基于"drill"构建，可运行真实的 Claude Code、Codex 和 Gemini 会话并使用 LLM 进行评判。多个 bash 测试套件在有更严格的 drill 场景覆盖后退役；少数没有等效场景的被保留。从此以后，`tests/` 存放插件代码测试，`evals/` 存放技能行为测试，`docs/testing.md` 解释这一划分。新后端覆盖 Antigravity、Pi 和更多模型，新的 shell-lint 和 pre-commit 检查保护 harness。(#1541)

### Bug 修复

- **systematic-debugging 不再强制每个会话进入扩展思考模式。** 一个项目包含了 Claude Code 扫描的确切关键字，静默地在每次加载该技能的会话中触发开关。用一个连字符断开关键字；文本仍然可读。(#1283, by @Nick Galatis)
- **Windows SessionStart hook 不再在每个会话中打印写入错误**——每个 `printf` 现在通过 `cat` 路由以吸收 broken pipe，输出在其他方面不变。(#1612, reported by @silvertakana)
- **Windows 前台模式**跟踪正确的进程并在 MSYS2 上清除其所有者 PID。(by @nestorluiscamachopaz)
- **`using-superpowers` bootstrap** 不再列出"debugging"作为一项不存在的技能。(reported by @mhat)
- **TDD 技能**链接了测试反模式参考。(#1532, #1529; link fix #1474 by @Stable Genius)
- **`using-git-worktrees`** 修复了步骤编号并删除了过时的 Cursor 引用。(#1522, and by @fuleinist)
- **Codex 审查技能**将私人内部玩笑替换为普通指导。(#1531)

### 文档和贡献者指南

- **将 Superpowers 移植到新 harness 的指南**（`docs/porting-to-a-new-harness.md`）列出了每个集成需要的三个部分以及成败关键的一条规则：在会话启动时加载 bootstrap。
- **每个 PR 和 issue 现在披露其制作方式**——模型、harness、版本和已安装的插件，或注明是手工编写的。我们根据产生贡献的内容对其有不同的评判标准。PR 也以 `dev` 而不是 `main` 为目标。PR 模板、所有三个 issue 模板以及新的 platform-support 模板都包含了这一要求。

### 贡献者

感谢 @mattvanhorn, @nawfal, @Nick Galatis, @silvertakana, @nestorluiscamachopaz, @qer, @mhat, @Stable Genius, @fuleinist, @dev_Hakaze, @robotsnh, Rahul, and @arittr。

## v5.1.0 (2026-04-30)

### 移除

- **旧的 slash 命令已移除**——`/brainstorm`、`/execute-plan` 和 `/write-plan` 已删除。它们只是已弃用的存根，除了告诉用户调用对应技能外什么也不做。请直接调用 `superpowers:brainstorming`、`superpowers:executing-plans` 和 `superpowers:writing-plans`。(#1188)
- **`superpowers:code-reviewer` 命名智能体已移除**——该智能体是插件唯一的命名智能体，仅被两个技能使用，而仓库中的所有其他审查/实施子智能体都通过 `general-purpose` 与提示模板一起分派。该智能体的人设和检查清单已合并到 `skills/requesting-code-review/code-reviewer.md` 中，作为自包含的 Task 分派模板。任何分派 `Task (superpowers:code-reviewer)` 的应改为使用 `Task (general-purpose)` 加提示模板。(PR #1299)
- **从技能中移除了集成部分**——这些是智能体尚未拥有原生技能系统时期的遗留物，对引导没有帮助。

### Worktree 技能重写

`using-git-worktrees` 和 `finishing-a-development-branch` 现在检测智能体是否已在隔离 worktree 中运行，并优先使用 harness 的原生 worktree 控件，然后才回退到 `git worktree`。行为经过 TDD 验证，并在五个 harness 上进行了跨平台检查。(PRI-974, PR #1121)

- **环境检测**——两个技能在做任何事之前检查 `GIT_DIR != GIT_COMMON`；如果已在关联 worktree 中，则完全跳过创建。子模块防护防止错误检测。
- **创建 worktree 前征求同意**——`using-git-worktrees` 不再隐式创建 worktree；技能首先询问用户。修复 #991（subagent-driven-development 在未经同意的情况下自动创建 worktree）。
- **原生工具偏好（步骤 1a）**——当 harness 暴露自己的 worktree 工具（如 Codex），技能优先使用它。用户明确表达的偏好会被尊重。
- **基于来源的清理**——`finishing-a-development-branch` 仅清理 `.worktrees/` 内的 worktree（由 superpowers 创建）；之外的任何内容均不受影响。修复 #940（Option 2 错误地清理了 worktree）、#999（合并后移除的顺序问题）和 #238（在 `git worktree remove` 前 `cd` 到仓库根目录）。
- **Detached HEAD 处理**——当没有分支可供合并时，完成菜单折叠为两个选项。
- **技能示例中的硬编码 `/Users/jesse` 路径**替换为通用占位符。(#858, PR #1122)

### AI 智能体贡献者指南

`CLAUDE.md` 顶部的两个新部分（符号链接到 `AGENTS.md`）直接面向 AI 智能体。对过去 100 个关闭的 PR 的审计显示，94% 的拒绝率是由 AI 生成的垃圾驱动的：智能体未阅读 PR 模板、提交了重复内容、编造了问题描述，或将 fork 或领域特定的改动推到了上游。

- **提交前检查清单**——阅读 PR 模板、搜索已有 PR、验证存在真实问题、确认改动属于核心、在提交前向人类伙伴展示完整 diff。
- **我们不接受的内容**——第三方依赖、对技能内容的"合规"重写、项目特定配置、批量 PR、推测性修复、领域特定技能、fork 特定改动、编造的内容和捆绑不相关的改动。
- **新 harness PR 需要会话记录**——大多数过去的新 harness 集成复制了技能文件或用 `npx skills` 包装，而不是在会话启动时加载 `using-superpowers` bootstrap。现在要求提供验收测试（"Let's make a react todo list"必须在干净会话中自动触发 `brainstorming`）和完整记录。

### Codex 插件镜像工具

新的 `sync-to-codex-plugin` 脚本将 superpowers 镜像到 OpenAI Codex 插件市场，名称为 `prime-radiant-inc/openai-codex-plugins`。路径/用户无关，任何团队成员都可以运行。(PR #1165)

- 每次运行时在临时目录中全新 clone fork，内联重新生成覆盖层，并提交 PR；从脚本自身位置自动检测上游，并预检 `rsync`/`git`/`gh auth`/`python3`。
- `--bootstrap` 标志用于首次设置；`EXCLUDES` 模式锚定在源根目录；`assets/` 被排除。
- 镜像 `CODE_OF_CONDUCT.md`；丢弃 `agents/openai.yaml` 覆盖层。
- 在镜像的 `plugin.json` 中植入 `interface.defaultPrompt`。(PR #1180 by @arittr)
- Codex 插件文件被提交到源仓库，以便同步脚本使用规范版本；Codex 市场元数据被保留。

### OpenCode

- **Bootstrap 内容在模块级缓存**——`getBootstrapContent()` 在每次智能体步骤中调用 `fs.existsSync` + `fs.readFileSync` + frontmatter 正则（`experimental.chat.messages.transform` hook 在 OpenCode 的智能体循环中每一步都触发）。现在只读取一次，缓存整个会话生命周期，对文件缺失情况设有 null 哨兵值。15 个回归测试覆盖缓存行为、fs 调用数、注入防护、缺失文件哨兵和缓存重置。（修复 #1202）
- **集成测试现代化。**
- **README 中明确了安装注意事项。**

### 代码审查整合

`requesting-code-review` 现在是自包含的：人设、检查清单和分派模板位于 `skills/requesting-code-review/code-reviewer.md`，技能直接分派 `Task (general-purpose)`。(PR #1299)

- **单一真实来源**——之前存在于 `agents/code-reviewer.md` 和技能的占位符模板中（且各自独立漂移）的人设/检查清单现在为一个文件。
- **`subagent-driven-development` 跟进**——其 `code-quality-reviewer-prompt.md` 现在分派 `Task (general-purpose)` 而不是命名智能体。
- **添加了行为测试**——`tests/claude-code/test-requesting-code-review.sh` 在一个小项目中植入真实 bug（SQL 注入、明文密码处理、凭证日志记录），并断言分派的审查者以 Critical/Important 严重度标记每个植入的问题并拒绝批准 diff。

> 注意：`tests/claude-code/test-requesting-code-review.sh` 和 `tests/claude-code/test-document-review-system.sh`（本文后面提到）于 2026-05-06 升格为 drill 场景并从 `tests/` 中移除。参见 `evals/scenarios/code-review-catches-planted-bugs.yaml` 和 `evals/scenarios/spec-reviewer-catches-planted-flaws.yaml`。上述及下述引用作为本节所描述工作的日期化产物保留。
- **Codex 和 Copilot 变通方案文档被精简**——`references/codex-tools.md` 和 `references/copilot-tools.md` 中的"Named agent dispatch"部分记录了如何将命名智能体展平为通用分派。由于不再附带命名智能体，这些变通方案不再必要；两个部分均已删除。

### 子智能体驱动开发

- **不再每 3 个任务暂停**——`requesting-code-review` 中的"review after each batch (3 tasks)"节奏（最初用于 `executing-plans`）泄漏到了 `subagent-driven-development`。替换为"每个任务或自然检查点"加上一条显式的连续执行指令。
- **SDD 集成测试现在运行其断言**——三个独立的 bug 导致测试在打印任何验证结果之前静默退出：工作目录路径中未解析的 `..` 段、`find | sort | head -1` 与 `set -euo pipefail` 的交互（生产者上的 SIGPIPE 杀死了脚本），以及 `claude -p` 调用中缺少 `--plugin-dir`，导致测试加载了已安装的插件而不是工作树。三个问题均已修复；六个验证测试现在实际针对真实的端到端 SDD 运行执行。

### Cursor

- **Windows SessionStart hook** 通过 `run-hook.cmd` 路由，而不是直接调用无扩展名的 `session-start` 脚本。修复了 Windows 在编辑器中打开文件而非运行它的错误。同时从 `hooks-cursor.json` 中删除了意外的 UTF-8 BOM。

### Gemini CLI

- **子智能体分派映射**——Gemini 的 `Task` 分派现在映射到 `@agent-name` / `@generalist`，并行子智能体分派已为独立任务文档化。

### 技能

- **技能内容中的术语清理。**

### 文档和安装

- **Factory Droid 安装说明**已添加到 README。
- **README 中的快速开始安装链接。**(PR #1293 by @arittr)
- **Codex 插件安装指导**已更新。(PR #1288 by @arittr)
- **工具参考中 Codex `wait` 映射**更正为 `wait_agent`。
- **安装顺序重新组织**；Codex 安装说明已清理。
- **移除了残留的 `CHANGELOG.md`**，以 `RELEASE-NOTES.md` 作为单一来源。(PR #1163 by @shaanmajid)
- **Discord 邀请链接**已修复；发布公告链接和详细的 Discord 描述已添加到社区部分。

### 社区

- @shaanmajid — 残留 `CHANGELOG.md` 的移除 (PR #1163)
- @arittr — README 快速开始安装链接 (#1293)、Codex 插件安装指导 (#1288)、`sync-to-codex-plugin` `interface.defaultPrompt` 植入 (#1180)

## v5.0.7 (2026-03-31)

### GitHub Copilot CLI 支持

- **SessionStart 上下文注入**——Copilot CLI v1.0.11 增加了对 sessionStart hook 输出中 `additionalContext` 的支持。session-start hook 现在检测 `COPILOT_CLI` 环境变量并发出 SDK 标准的 `{ "additionalContext": "..." }` 格式，为 Copilot CLI 用户提供完整的 superpowers bootstrap。(Original fix by @culinablaz in PR #910)
- **工具映射**——添加了 `references/copilot-tools.md`，包含完整的 Claude Code 到 Copilot CLI 工具等价表
- **技能和 README 更新**——将 Copilot CLI 添加到 `using-superpowers` 技能的平台说明和 README 安装部分

### OpenCode 修复

- **技能路径一致性**——bootstrap 文本不再宣传一个与运行时路径不匹配的误导性 `configDir/skills/superpowers/` 路径。智能体应使用原生 `skill` 工具，而不是按路径导航到文件。测试现在使用从单一真实来源派生的统一路径。(#847, #916)
- **Bootstrap 作为用户消息**——将 bootstrap 注入从 `experimental.chat.system.transform` 移动到 `experimental.chat.messages.transform`，在第一条用户消息前添加而不是添加系统消息。避免了每轮重复系统消息带来的 token 膨胀 (#750)，并修复了与 Qwen 及其他在多个系统消息上出问题的模型的兼容性 (#894)。

## v5.0.6 (2026-03-24)

### 内联自审查替换子智能体审查循环

子智能体审查循环（分派新的智能体来审查计划/规范）使执行时间翻倍（约 25 分钟开销），而没有可测量的计划质量提升。跨 5 个版本、每个 5 次试验的回归测试显示，无论审查循环是否运行，质量分数都相同。

- **brainstorming**——用内联 Spec Self-Review 检查清单替换了 Spec Review Loop（子智能体分派 + 3 次迭代上限）：占位符扫描、内部一致性、范围检查、歧义检查
- **writing-plans**——用内联 Self-Review 检查清单替换了 Plan Review Loop（子智能体分派 + 3 次迭代上限）：规范覆盖、占位符扫描、类型一致性
- **writing-plans**——添加了显式的"No Placeholders"部分，定义了计划失败的标准（TBD、模糊描述、未定义引用、"similar to Task N"）
- 自审查在约 30 秒内捕获每次运行 3-5 个真实 bug，而非约 25 分钟，缺陷率与子智能体方法相当

### Brainstorm 服务器

- **会话目录重构**——brainstorm 服务器的会话目录现在包含两个对等的子目录：`content/`（在浏览器中提供的 HTML 文件）和 `state/`（事件、server-info、pid、日志）。此前，服务器状态和用户交互数据与提供的文件存储在同一位置，使其可通过 HTTP 访问。`screen_dir` 和 `state_dir` 路径都包含在 server-started JSON 中。（Reported by 吉田仁）

### Bug 修复

- **Owner-PID 生命周期修复**——brainstorm 服务器的 owner-PID 监控有两个 bug 导致在 60 秒内误关闭：(1) 跨用户 PID（Tailscale SSH 等）的 EPERM 被视为"进程已死"，(2) 在 WSL 上，祖父进程 PID 解析为一个在首次生命周期检查前退出的小生命周期子进程。修复方法是将 EPERM 视为"alive"并在启动时验证 owner PID——如果它已经死亡，则禁用监控，服务器依赖 30 分钟空闲超时。这也从 `start-server.sh` 中移除了 Windows/MSYS2 特定例外，因为服务器现在以通用方式处理。(#879)
- **writing-skills**——纠正了 SKILL.md frontmatter 只支持"only two fields"的错误说法；现在说明是"two required fields"并链接到 agentskills.io 规范以了解所有支持的字段 (PR #882 by @arittr)

### Codex App 兼容性

- **codex-tools**——添加了命名智能体分派映射，记录如何将 Claude Code 的命名智能体类型转换为 Codex 带 worker 角色的 `spawn_agent` (PR #647 by @arittr)
- **codex-tools**——添加了环境检测和 Codex App 完成部分，用于 worktree 感知技能 (by @arittr)
- **设计规范**——添加了 Codex App 兼容性设计规范 (PRI-823)，涵盖只读环境检测、worktree 安全的技能行为和沙箱回退模式 (by @arittr)

## v5.0.5 (2026-03-17)

### Bug 修复

- **Brainstorm 服务器 ESM 修复**——将 `server.js` 重命名为 `server.cjs`，使得在 Node.js 22+ 上根 `package.json` 的 `"type": "module"` 导致 `require()` 失败时，brainstorming 服务器能够正常启动。(PR #784 by @sarbojitrana, fixes #774, #780, #783)
- **Windows 上的 Brainstorm owner-PID**——在 Windows/MSYS2 上跳过 PID 生命周期监控，因为 PID 命名空间对 Node.js 不可见，会阻止服务器在 60 秒后自我终止。(#770, docs from PR #768 by @lucasyhzlu-debug)
- **stop-server.sh 可靠性**——在报告成功之前验证服务器进程确实已死亡。SIGTERM + 2s 等待 + SIGKILL fallback。(#723)

### 变更

- **执行交接**——恢复用户在编写计划后选择子智能体驱动或内联执行的选项。推荐子智能体驱动但不再是强制的。

## v5.0.4 (2026-03-16)

### 审查循环细化

通过消除不必要的审查轮次和缩紧审查者关注范围，显著减少 token 使用并加快了规范和计划审查。

- **单一全计划审查**——计划审查者现在一次性审查完整计划，而不是逐块审查。移除了所有块相关概念（`## Chunk N:` 标题、1000 行块限制、每块分派）。
- **提高了阻塞问题的标准**——规范和计划审查者提示现在都包含"Calibration"部分：只标记在实施过程中会造成实际问题的内容。小措辞问题、风格偏好和格式吹毛求疵不应阻止批准。
- **减少最大审查迭代次数**——对规范和计划审查循环，从 5 次降至 3 次。如果审查者校准正确，3 轮就已经足够了。
- **精简审查者检查清单**——规范审查者从 7 个类别减至 5 个；计划审查者从 7 个减至 4 个。移除了以格式为核心的检查（任务语法、块大小），改为关注实质（可构建性、规范对齐）。

### OpenCode

- **一行插件安装**——OpenCode 插件现在通过 `config` hook 自动注册技能目录。不需要符号链接或 `skills.paths` 配置。安装只需在 `opencode.json` 中添加一行。(PR #753)
- **添加了 `package.json`**，以便 OpenCode 可以从 git 将 superpowers 作为 npm 包安装。

### Bug 修复

- **验证服务器确实已停止**——`stop-server.sh` 现在在报告成功之前确认进程已死亡。SIGTERM + 2s 等待 + SIGKILL fallback。如果进程存活则报告失败。(PR #751)
- **通用智能体语言**——brainstorm 伴侣的等待页面现在说"the agent"而不是"Claude"。

## v5.0.3 (2026-03-15)

### Cursor 支持

- **Cursor hooks**——添加了 `hooks/hooks-cursor.json`，使用 Cursor 的 camelCase 格式（`sessionStart`、`version: 1`），并更新了 `.cursor-plugin/plugin.json` 以引用它。修复了 `session-start` 中的平台检测，优先检查 `CURSOR_PLUGIN_ROOT`（Cursor 也可能设置 `CLAUDE_PLUGIN_ROOT`）。(Based on PR #709)

### Bug 修复

- **停止在 `--resume` 上触发 SessionStart hook**——启动 hook 在恢复的会话中重新注入上下文，而这些会话已经在对话历史中拥有上下文。现在 hook 仅在 `startup`、`clear` 和 `compact` 上触发。
- **Bash 5.3+ hook 挂起**——在 `hooks/session-start` 中将 heredoc（`cat <<EOF`）替换为 `printf`。修复了 macOS 上使用 Homebrew bash 5.3+ 时，由 bash 在 heredoc 中大变量展开的回归问题导致的无限期挂起。(#572, #571)
- **POSIX 安全的 hook 脚本**——在 `hooks/session-start` 中将 `${BASH_SOURCE[0]:-$0}` 替换为 `$0`。修复了在 Ubuntu/Debian 上 `/bin/sh` 是 dash 时的"Bad substitution"错误。(#553)
- **可移植的 shebangs**——将所有 shell 脚本中的 `#!/bin/bash` 替换为 `#!/usr/bin/env bash`。修复了在 NixOS、FreeBSD 和带有 Homebrew bash 的 macOS 上 `/bin/bash` 过时或缺失时无法执行的问题。(#700)
- **Windows 上的 Brainstorm 服务器**——自动检测 Windows/Git Bash（`OSTYPE=msys*`、`MSYSTEM`）并切换到前台模式，修复了由 `nohup`/`disown` 进程回收导致的静默服务器失败。(#737)
- **Codex 文档修复**——在 Codex 文档中将已弃用的 `collab` 标志替换为 `multi_agent`。(PR #749)

## v5.0.2 (2026-03-11)

### 零依赖 Brainstorm 服务器

**移除了所有 vendored node_modules——server.js 现在完全自包含**

- 用使用内置 `http`、`fs` 和 `crypto` 模块的零依赖 Node.js 服务器替换了 Express/Chokidar/WebSocket 依赖
- 移除了约 1,200 行的 vendored `node_modules/`、`package.json` 和 `package-lock.json`
- 自定义 WebSocket 协议实现（RFC 6455 帧、ping/pong、正确的关闭握手）
- 原生 `fs.watch()` 文件监视替换了 Chokidar
- 完整测试套件：HTTP 服务、WebSocket 协议、文件监视和集成测试

### Brainstorm 服务器可靠性

- **空闲 30 分钟后自动退出**——服务器在无客户端连接时关闭，防止孤立进程
- **所有者进程跟踪**——服务器监控父 harness PID，并在所属会话死亡时退出
- **活跃性检查**——技能在重用现有实例前验证服务器响应正常
- **编码修复**——在提供的 HTML 页面上正确的 `<meta charset="utf-8">`

### 子智能体上下文隔离

- 所有委派技能（brainstorming、dispatching-parallel-agents、requesting-code-review、subagent-driven-development、writing-plans）现在包含上下文隔离原则
- 子智能体仅接收其需要的上下文，防止上下文窗口污染

## v5.0.1 (2026-03-10)

### Agentskills 合规

**Brainstorm-server 移至技能目录内**

- 将 `lib/brainstorm-server/` 移至 `skills/brainstorming/scripts/`，遵循 [agentskills.io](https://agentskills.io) 规范
- 所有 `${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/` 引用替换为相对 `scripts/` 路径
- 技能现在完全跨平台可移植——无需平台特定的环境变量来定位脚本
- `lib/` 目录已移除（之前是最后剩余的内容）

### 新功能

**Gemini CLI 扩展**

- 通过仓库根目录的 `gemini-extension.json` 和 `GEMINI.md` 提供原生 Gemini CLI 扩展支持
- `GEMINI.md` 在会话启动时 @imports `using-superpowers` 技能和工具映射表
- Gemini CLI 工具映射参考（`skills/using-superpowers/references/gemini-tools.md`）——将 Claude Code 工具名称（Read、Write、Edit、Bash 等）转换为 Gemini CLI 等价物（read_file、write_file、replace 等）
- 文档化 Gemini CLI 限制：不支持子智能体，技能回退到 `executing-plans`
- 扩展根目录位于仓库根目录，以实现跨平台兼容性（避免 Windows 符号链接问题）
- 安装说明已添加到 README

### 改进

**多平台 brainstorm 服务器启动**

- visual-companion.md 中的每个平台启动说明：Claude Code（默认模式）、Codex（通过 `CODEX_CI` 自动前台）、Gemini CLI（带 `is_background` 的 `--foreground`）以及其他环境的回退
- 服务器现在将启动 JSON 写入 `$SCREEN_DIR/.server-info`，以便智能体在 stdout 被后台执行为隐藏时仍能找到 URL 和端口

**Brainstorm 服务器依赖已捆绑**

- `node_modules` vendored 到仓库中，使 brainstorm 服务器在全新插件安装时立即工作，无需运行时安装 `npm`
- 从捆绑依赖中移除了 `fsevents`（仅 macOS 原生二进制文件；chokidar 在没有它的情况下优雅回退）
- 如果 `node_modules` 缺失，回退自动安装通过 `npm install`

**OpenCode 工具映射修复**

- `TodoWrite` → `todowrite`（之前错误映射到 `update_plan`）；已对照 OpenCode 源代码验证

### Bug 修复

**Windows/Linux：单引号破坏 SessionStart hook** (#577, #529, #644, PR #585)

- hooks.json 中 `${CLAUDE_PLUGIN_ROOT}` 周围的单引号在 Windows 上失败（cmd.exe 不识别单引号作为路径分隔符）和在 Linux 上失败（单引号阻止变量展开）
- 修复：将单引号替换为转义的双引号——在 macOS bash、Windows cmd.exe、Windows Git Bash 和 Linux 上均工作正常，路径带或不带空格均可
- 已在 Windows 11 (NT 10.0.26200.0) 上使用 Claude Code 2.1.72 和 Git for Windows 验证

**Brainstorming 规范审查循环被跳过** (#677)

- 规范审查循环（分派 spec-document-reviewer 子智能体，迭代直到批准）存在于"After the Design"部分的散文中，但缺失于检查清单和流程图中
- 由于智能体遵循流程图和检查清单比遵循散文更可靠，规范审查步骤被完全跳过
- 将第 7 步（规范审查循环）添加到检查清单，并向 dot 图中添加了相应节点
- 使用 `claude --plugin-dir` 和 `claude-session-driver` 测试：worker 现在正确分派审查者

**Cursor 安装命令** (PR #676)

- 修复了 README 中的 Cursor 安装命令：`/plugin-add` → `/add-plugin`（经 Cursor 2.5 发布公告确认）

**brainstorming 中的用户审查门控** (#565)

- 在规范完成和 writing-plans 交接之间添加了显式的用户审查步骤
- 在实施计划开始之前，用户必须批准规范
- 检查清单、流程和散文更新了新的门控

**Session-start hook 每个平台只发出一次上下文**

- Hook 现在检测是运行在 Claude Code 还是其他平台
- 为 Claude Code 发出 `hookSpecificOutput`，为其他平台发出 `additional_context`——防止双重上下文注入

**token 分析脚本中的 Linting 修复**

- `tests/claude-code/analyze-token-usage.py` 中 `except:` → `except Exception:`

### 维护

**移除了死代码**

- 删除了 `lib/skills-core.js` 及其测试（`tests/opencode/test-skills-core.js`）——自 2026 年 2 月起未使用
- 从 `tests/opencode/test-plugin-loading.sh` 中移除了 skills-core 存在性检查

### 社区

- @karuturi — Claude Code 官方市场安装说明 (PR #610)
- @mvanhorn — session-start hook 双重发出修复、OpenCode 工具映射修复
- @daniel-graham — bare except 的 linting 修复
- PR #585 作者 — Windows/Linux hooks 引用修复

---

## v5.0.0 (2026-03-09)

### 不兼容变更

**规范和计划目录重构**

- 规范（brainstorming 输出）现在保存到 `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`
- 计划（writing-plans 输出）现在保存到 `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- 用户对规范/计划位置的偏好覆盖这些默认值
- 所有内部技能引用、测试文件和示例路径已更新以匹配
- 迁移：如果需要，将现有文件从 `docs/plans/` 移动到新位置

**在具有子智能体能力的 harness 上子智能体驱动开发为强制性**

Writing-plans 不再提供子智能体驱动和 executing-plans 之间的选择。在支持子智能体的 harness 上（Claude Code、Codex），superpowers:subagent-driven-development 是必需的。Executing-plans 保留给没有子智能体能力的 harness，现在告诉用户 Superpowers 在支持子智能体的平台上效果更好。

**Executing-plans 不再批量执行**

移除了"execute 3 tasks then stop for review"模式。计划现在持续执行，仅在遇到阻塞时停止。

**Slash 命令已弃用**

`/brainstorm`、`/write-plan` 和 `/execute-plan` 现在显示弃用通知，将用户引导到相应的技能。命令将在下一个主要版本中移除。

### 新功能

**可视化 brainstorming 伴侣**

为 brainstorming 会话提供的可选浏览器伴侣。当某个主题适合视觉展示时，brainstorming 技能会提议在终端对话旁边的浏览器窗口中展示原型、图表、对比和其他内容。

- `lib/brainstorm-server/`——WebSocket 服务器，带有浏览器辅助库、会话管理脚本，以及暗色/亮色主题的框架模板（"Superpowers Brainstorming"，带 GitHub 链接）
- `skills/brainstorming/visual-companion.md`——服务器工作流、屏幕创作和反馈收集的渐进式披露指南
- Brainstorming 技能在其流程中增加了一个可视化伴侣决策点：在探索项目上下文后，技能评估接下来问题是否涉及视觉内容，并在自己的消息中提供伴侣选项
- 每个问题的决策：即使在接受后，每个问题也会评估浏览器还是终端更合适
- `tests/brainstorm-server/` 中的集成测试

**文档审查系统**

使用子智能体分派进行规范和计划文件的自动审查循环：

- `skills/brainstorming/spec-document-reviewer-prompt.md`——审查者检查完整性、一致性、架构和 YAGNI
- `skills/writing-plans/plan-document-reviewer-prompt.md`——审查者检查规范对齐、任务分解、文件结构和文件大小
- Brainstorming 在编写设计文档后分派规范审查者
- Writing-plans 在每个部分后包含基于块的计划审查循环
- 审查循环重复直到批准，或在 5 次迭代后升级
- `tests/claude-code/test-document-review-system.sh` 中的端到端测试
- `docs/superpowers/` 中的设计规范和实施计划

**整个技能管线中的架构指导**

为 brainstorming、writing-plans 和 subagent-driven-development 添加了面向隔离的设计和文件大小意识指导：

- **Brainstorming**——新部分："Design for isolation and clarity"（清晰边界、明确定义的接口、独立可测试的单元）和"Working in existing codebases"（遵循现有模式、仅做有针对性的改进）
- **Writing-plans**——新"File Structure"部分：在定义任务之前规划文件和职责。新"Scope Check"后盾：捕获应在 brainstorming 阶段分解的多子系统规范
- **SDD 实施者**——新"Code Organization"部分（遵循计划的文件结构、报告对文件增长的担忧）和"When You're in Over Your Head"升级指导
- **SDD 代码质量审查者**——现在检查架构、单元分解、计划合规性和文件增长
- **规范/计划审查者**——架构和文件大小添加到审查标准
- **范围评估**——Brainstorming 现在评估项目是否太大而不适合单个规范。多子系统请求早期标记并分解为子项目，每个都有其自己的规范→计划→实施周期

**子智能体驱动开发改进**

- **模型选择**——按任务类型选择模型能力的指导：廉价模型用于机械实施、标准模型用于集成、强模型用于架构和审查
- **实施者状态协议**——子智能体现在报告 DONE、DONE_WITH_CONCERNS、BLOCKED 或 NEEDS_CONTEXT。控制器适当处理每种状态：使用更多上下文重新分派、升级模型能力、拆分任务，或升级给人类

### 改进

**指令优先级层次**

在 using-superpowers 中添加了显式的优先级顺序：

1. 用户的显式指令（CLAUDE.md、AGENTS.md、直接请求）——最高优先级
2. Superpowers 技能——覆盖默认系统行为
3. 默认系统提示——最低优先级

如果 CLAUDE.md 或 AGENTS.md 说"don't use TDD"而技能说"always use TDD"，用户的指令胜出。

**SUBAGENT-STOP 门控**

在 using-superpowers 中添加了 `<SUBAGENT-STOP>` 块。为特定任务分派的子智能体现在跳过技能，而不是激活 1% 规则并调用完整的技能工作流。

**多平台改进**

- Codex 工具映射移至渐进式披露参考文件（`references/codex-tools.md`）
- 添加了 Platform Adaptation 指针，以便非 Claude Code 平台可以找到工具等价物
- 计划标题现在称呼"agentic workers"而不是特定的"Claude"
- Collab 功能要求在 `docs/README.codex.md` 中文档化

**Writing-plans 模板更新**

- 计划步骤现在使用复选框语法（`- [ ] **Step N:**`）用于进度跟踪
- 计划标题同时引用 subagent-driven-development 和 executing-plans，带有平台感知路由

---

## v4.3.1 (2026-02-21)

### 新增

**Cursor 支持**

Superpowers 现在使用 Cursor 的插件系统。包括 `.cursor-plugin/plugin.json` 清单和 README 中 Cursor 特定的安装说明。SessionStart hook 输出现在在现有的 `hookSpecificOutput.additionalContext` 旁边包含 `additional_context` 字段，以实现 Cursor hook 兼容性。

### 已修复

**Windows：恢复了 polyglot wrapper 以实现可靠的 hook 执行 (#518, #504, #491, #487, #466, #440)**

Claude Code 在 Windows 上的 `.sh` 自动检测会在 hook 命令前加上 `bash`，破坏执行。修复方法：

- 将 `session-start.sh` 重命名为 `session-start`（无扩展名），使自动检测不会干扰
- 恢复了 `run-hook.cmd` polyglot wrapper，带有多位置 bash 发现（标准 Git for Windows 路径，然后 PATH fallback）
- 如果没有找到 bash，静默退出而不是报错
- 在 Unix 上，wrapper 通过 `exec bash` 直接运行脚本
- 使用 POSIX 安全的 `dirname "$0"` 路径解析（在 dash/sh 上工作，不只是 bash）

这修复了 Windows 上带空格的路径、缺失 WSL、MSYS 上的 `set -euo pipefail` 脆弱性和反斜杠损坏导致的 SessionStart 失败。

## v4.3.0 (2026-02-12)

此修复应显著提高 superpowers 技能合规性，并应减少 Claude 无意中进入其原生 plan mode 的机会。

### 变更

**Brainstorming 技能现在强制执行其工作流而不是描述它**

模型会跳过设计阶段直接跳转到实施技能（如 frontend-design），或将整个 brainstorming 过程折叠为单个文本块。该技能现在使用硬性门控、强制性检查清单和 graphviz 流程图来强制执行合规性：

- `<HARD-GATE>`：在呈现设计并获得用户批准之前，不允许实施技能、代码或脚手架
- 显式检查清单（6 项），必须作为任务创建并按顺序完成
- Graphviz 流程图，`writing-plans` 作为唯一有效的终止状态
- 针对"this is too simple to need a design"的反模式标注——这正是模型用来跳过流程的理性化方式
- 设计部分大小基于部分复杂度而不是项目复杂度

**Using-superpowers 工作流图拦截 EnterPlanMode**

在技能流程图中添加了 `EnterPlanMode` 拦截。当模型即将进入 Claude 的原生 plan mode 时，它会检查 brainstorming 是否已经发生，并通过 brainstorming 技能路由。Plan mode 不会被进入。

### 已修复

**SessionStart hook 现在同步运行**

在 hooks.json 中将 `async: true` 改为 `async: false`。当异步时，hook 可能在模型的第一轮之前无法完成，意味着 using-superpowers 指令不在第一条消息的上下文中。

## v4.2.0 (2026-02-05)

### 不兼容变更

**Codex：用原生技能发现替换了 bootstrap CLI**

`superpowers-codex` bootstrap CLI、Windows `.cmd` wrapper 和相关的 bootstrap 内容文件已被移除。Codex 现在通过 `~/.agents/skills/superpowers/` 符号链接使用原生技能发现，所以旧的 `use_skill`/`find_skills` CLI 工具不再需要。

安装现在只是 clone + symlink（文档化在 INSTALL.md 中）。不需要 Node.js 依赖。旧的 `~/.codex/skills/` 路径已弃用。

### 修复

**Windows：修复了 Claude Code 2.1.x hook 执行 (#331)**

Claude Code 2.1.x 改变了 Windows 上 hooks 的执行方式：它现在自动检测命令中的 `.sh` 文件并在前面添加 `bash`。这破坏了 polyglot wrapper 模式，因为 `bash "run-hook.cmd" session-start.sh` 会尝试将 .cmd 文件作为 bash 脚本执行。

修复：hooks.json 现在直接调用 session-start.sh。Claude Code 2.1.x 自动处理 bash 调用。同时添加了 .gitattributes 以强制 shell 脚本使用 LF 行尾（修复 Windows checkout 上的 CRLF 问题）。

**Windows：SessionStart hook 异步运行以防止终端冻结 (#404, #413, #414, #419)**

同步 SessionStart hook 阻止 TUI 在 Windows 上进入 raw mode，冻结所有键盘输入。异步运行 hook 防止冻结，同时仍然注入 superpowers 上下文。

**Windows：修复了 O(n^2) 的 `escape_for_json` 性能**

使用 `${input:$i:1}` 的逐字符循环在 bash 中是 O(n^2)，因为子字符串复制开销。在 Windows Git Bash 上这需要 60 多秒。替换为 bash 参数替换（`${s//old/new}`），将每个模式作为单次 C 级别遍历运行——在 macOS 上快 7 倍，在 Windows 上快得多。

**Codex：修复了 Windows/PowerShell 调用 (#285, #243)**

- Windows 不识别 shebangs，因此直接调用无扩展名的 `superpowers-codex` 脚本触发了"Open with"对话框。所有调用现在以 `node` 为前缀。
- 修复了 Windows 上的 `~/` 路径展开——PowerShell 在作为参数传递给 `node` 时不展开 `~`。改为 `$HOME`，在 bash 和 PowerShell 中都能正常展开。

**Codex：修复了安装程序中的路径解析**

使用 `fileURLToPath()` 而不是手动 URL 路径名解析，在所有平台上正确处理带空格和特殊字符的路径。

**Codex：修复了 writing-skills 中过时的技能路径**

将已弃用的 `~/.codex/skills/` 引用更新为 `~/.agents/skills/` 用于原生发现。

### 改进

**实施前现在要求 worktree 隔离**

将 `using-git-worktrees` 添加为 `subagent-driven-development` 和 `executing-plans` 的必需技能。实施工作流现在明确要求在开始工作之前设置隔离的 worktree，防止直接在 main 上意外工作。

**主分支保护软化，要求显式同意**

技能不再完全禁止在主分支上工作，而是在用户明确同意后允许。更灵活同时仍确保用户了解其影响。

**简化的安装验证**

从验证步骤中移除了 `/help` 命令检查和特定 slash 命令列表。技能主要通过描述你想做什么来调用，而不是通过运行特定命令。

**Codex：在 bootstrap 中明确了子智能体工具映射**

改进了 Codex 工具如何映射到 Claude Code 等价物的文档，用于子智能体工作流。

### 测试

- 为 subagent-driven-development 添加了 worktree 要求测试
- 添加了主分支 red flag 警告测试
- 修复了技能识别测试断言中的大小写敏感性

---

## v4.1.1 (2026-01-23)

### 修复

**OpenCode：按照官方文档标准化为 `plugins/` 目录 (#343)**

OpenCode 的官方文档使用 `~/.config/opencode/plugins/`（复数）。我们的文档之前使用 `plugin/`（单数）。虽然 OpenCode 接受两种形式，但我们已经标准化为官方约定以避免混淆。

变更：
- 将仓库结构中的 `.opencode/plugin/` 重命名为 `.opencode/plugins/`
- 更新了所有平台的所有安装文档（INSTALL.md、README.opencode.md）
- 更新了测试脚本以匹配

**OpenCode：修复了符号链接说明 (#339, #342)**

- 在 `ln -s` 之前添加了显式的 `rm`（修复重新安装时的"file already exists"错误）
- 添加了 INSTALL.md 中缺失的 skills symlink 步骤
- 从已弃用的 `use_skill`/`find_skills` 更新为原生 `skill` 工具引用

---

## v4.1.0 (2026-01-23)

### 不兼容变更

**OpenCode：切换到原生技能系统**

Superpowers for OpenCode 现在使用 OpenCode 的原生 `skill` 工具，而不是自定义的 `use_skill`/`find_skills` 工具。这是一个更干净的集成，使用 OpenCode 内置的技能发现。

**需要迁移：** 技能必须符号链接到 `~/.config/opencode/skills/superpowers/`（参见更新后的安装文档）。

### 修复

**OpenCode：修复了会话启动时智能体重置 (#226)**

之前使用 `session.prompt({ noReply: true })` 的 bootstrap 注入方法导致 OpenCode 在第一条消息时将所选智能体重置为"build"。现在使用 `experimental.chat.system.transform` hook，直接修改系统提示而无副作用。

**OpenCode：修复了 Windows 安装 (#232)**

- 移除了对 `skills-core.js` 的依赖（消除文件被复制而不是符号链接时断裂的相对导入）
- 为 cmd.exe、PowerShell 和 Git Bash 添加了全面的 Windows 安装文档
- 文档化了每个平台上正确的 symlink 与 junction 使用

**Claude Code：修复了 Claude Code 2.1.x 的 Windows hook 执行**

Claude Code 2.1.x 改变了 Windows 上 hooks 的执行方式：它现在自动检测命令中的 `.sh` 文件并在前面添加 `bash`。这破坏了 polyglot wrapper 模式，因为 `bash "run-hook.cmd" session-start.sh` 会尝试将 .cmd 文件作为 bash 脚本执行。

修复：hooks.json 现在直接调用 session-start.sh。Claude Code 2.1.x 自动处理 bash 调用。同时添加了 .gitattributes 以强制 shell 脚本使用 LF 行尾（修复 Windows checkout 上的 CRLF 问题）。

---

## v4.0.3 (2025-12-26)

### 改进

**增强了 using-superpowers 技能以应对显式技能请求**

解决了一种失败模式：即使当用户显式按名称请求技能时（例如"subagent-driven-development, please"），Claude 也会跳过调用该技能。Claude 会想"I know what that means"并直接开始工作，而不是加载技能。

变更：
- 将"The Rule"更新为"Invoke relevant or requested skills"而不是"Check for skills"——强调主动调用而不是被动检查
- 添加了"BEFORE any response or action"——原始措辞只提到了"response"，但 Claude 有时会在没有先回复的情况下采取行动
- 添加了调用错误技能也没关系的保证——减少犹豫
- 添加了新的 red flag："I know what that means" → Knowing the concept ≠ using the skill

**添加了显式技能请求测试**

`tests/explicit-skill-requests/` 中的新测试套件，验证 Claude 在用户按名称请求技能时是否正确调用。包括单轮和多轮测试场景。

## v4.0.2 (2025-12-23)

### 修复

**Slash 命令现在仅限用户使用**

为所有三个 slash 命令（`/brainstorm`、`/execute-plan`、`/write-plan`）添加了 `disable-model-invocation: true`。Claude 不能再通过 Skill 工具调用这些命令——它们仅限于手动用户调用。

底层技能（`superpowers:brainstorming`、`superpowers:executing-plans`、`superpowers:writing-plans`）仍可供 Claude 自主调用。这一变化防止了 Claude 调用一个只是重定向到技能的命令时产生的混淆。

## v4.0.1 (2025-12-23)

### 修复

**明确了在 Claude Code 中访问技能的方式**

修复了一种令人困惑的模式，Claude 会通过 Skill 工具调用一个技能，然后尝试单独 Read 该技能文件。`using-superpowers` 技能现在明确说明 Skill 工具直接加载技能内容——无需读取文件。

- 向 `using-superpowers` 添加了"How to Access Skills"部分
- 将"read the skill"→"invoke the skill"在指令中
- 更新了 slash 命令使用完全限定的技能名称（例如 `superpowers:brainstorming`）

**向 receiving-code-review 添加了 GitHub 线程回复指导** (h/t @ralphbean)

添加了关于在原始线程中回复内联审查评论而不是作为顶级 PR 评论的注释。

**向 writing-skills 添加了自动化优先于文档的指导** (h/t @EthanJStark)

添加了指导：机械性约束应自动化而不是文档化——技能留给判断决策。

## v4.0.0 (2025-12-17)

### 新功能

**子智能体驱动开发中的两阶段代码审查**

子智能体工作流现在在每个任务后使用两个独立的审查阶段：

1. **规范合规性审查** - 怀疑态度的审查者验证实施是否精确匹配规范。捕获遗漏的需求和过度构建。不相信实施者的报告——读取实际代码。

2. **代码质量审查** - 仅在规范合规性通过后运行。审查代码清洁度、测试覆盖率、可维护性。

这捕获了代码写得很好但不匹配被要求内容的常见失败模式。审查是循环的，不是一次性的：如果审查者发现问题，实施者修复它们，然后审查者再次检查。

其他子智能体工作流改进：
- 控制器向 workers 提供完整任务文本（而非文件引用）
- Workers 可以在工作之前和期间提出澄清问题
- 在报告完成之前进行自我审查检查清单
- 计划在开始时读取一次，提取到 TodoWrite

`skills/subagent-driven-development/` 中的新提示模板：
- `implementer-prompt.md` - 包括自我审查检查清单，鼓励提问
- `spec-reviewer-prompt.md` - 针对需求的怀疑式验证
- `code-quality-reviewer-prompt.md` - 标准代码审查

**调试技术随工具统一**

`systematic-debugging` 现在捆绑了支持技术和工具：
- `root-cause-tracing.md` - 通过调用栈向后追踪 bug
- `defense-in-depth.md` - 在多个层添加验证
- `condition-based-waiting.md` - 用条件轮询替换任意超时
- `find-polluter.sh` - 二分脚本，找出哪个测试造成了污染
- `condition-based-waiting-example.ts` - 来自真实调试会话的完整实现

**测试反模式参考**

`test-driven-development` 现在包含 `testing-anti-patterns.md`，涵盖：
- 测试 mock 行为而不是真实行为
- 向生产类添加仅测试方法
- 在不理解依赖的情况下 mock
- 隐藏结构假设的不完整 mock

**技能测试基础设施**

三个新的测试框架用于验证技能行为：

`tests/skill-triggering/` - 验证技能从朴素提示中触发，无需显式命名。测试 6 个技能以确保仅凭描述就足够。

`tests/claude-code/` - 使用 `claude -p` 进行无头测试的集成测试。通过会话记录（JSONL）分析验证技能使用。包含 `analyze-token-usage.py` 用于成本跟踪。

`tests/subagent-driven-dev/` - 使用两个完整测试项目进行端到端工作流验证：
- `go-fractals/` - 带 Sierpinski/Mandelbrot 的 CLI 工具（10 个任务）
- `svelte-todo/` - 带 localStorage 和 Playwright 的 CRUD 应用（12 个任务）

### 主要变化

**DOT 流程图作为可执行规范**

使用 DOT/GraphViz 流程图重写了关键技能，将其作为权威流程定义。散文成为支持内容。

**描述陷阱**（记录在 `writing-skills` 中）：当描述包含工作流摘要时，发现技能描述会覆盖流程图内容。Claude 遵循简短描述而不是阅读详细的流程图。修复：描述必须是仅限触发（"Use when X"），不含流程细节。

**using-superpowers 中的技能优先级**

当多个技能适用时，流程技能（brainstorming、debugging）现在明确排在实施技能之前。"Build X"首先触发 brainstorming，然后是领域技能。

**brainstorming 触发器增强**

描述更改为命令式："You MUST use this before any creative work—creating features, building components, adding functionality, or modifying behavior."

### 不兼容变更

**技能整合** - 六个独立技能合并：
- `root-cause-tracing`、`defense-in-depth`、`condition-based-waiting` → 捆绑在 `systematic-debugging/` 中
- `testing-skills-with-subagents` → 捆绑在 `writing-skills/` 中
- `testing-anti-patterns` → 捆绑在 `test-driven-development/` 中
- `sharing-skills` 已移除（过时）

### 其他改进

- **render-graphs.js** - 从技能中提取 DOT 图表并渲染为 SVG 的工具
- **using-superpowers 中的 Rationalizations 表格** - 可扫描格式，包括新条目："I need more context first"、"Let me explore first"、"This feels productive"
- **docs/testing.md** - 使用 Claude Code 集成测试来测试技能的指南

---

## v3.6.2 (2025-12-03)

### 已修复

- **Linux 兼容性**：修复了 polyglot hook wrapper（`run-hook.cmd`）使用 POSIX 兼容语法
  - 在第 16 行将 bash 特定的 `${BASH_SOURCE[0]:-$0}` 替换为标准的 `$0`
  - 解决了 Ubuntu/Debian 系统上 `/bin/sh` 是 dash 时的"Bad substitution"错误
  - 修复 #141

---

## v3.5.1 (2025-11-24)

### 变更

- **OpenCode Bootstrap 重构**：从 `chat.message` hook 切换到 `session.created` 事件用于 bootstrap 注入
  - Bootstrap 现在通过 `session.prompt()` 以 `noReply: true` 在会话创建时注入
  - 明确告诉模型 using-superpowers 已经加载，以防止冗余的技能加载
  - 将 bootstrap 内容生成整合到共享的 `getBootstrapContent()` 辅助函数中
  - 更干净的单实现方法（移除了 fallback 模式）

---

## v3.5.0 (2025-11-23)

### 新增

- **OpenCode 支持**：OpenCode.ai 的原生 JavaScript 插件
  - 自定义工具：`use_skill` 和 `find_skills`
  - 在上下文压缩后保持技能持久性的消息插入模式
  - 通过 chat.message hook 自动上下文注入
  - 在 session.compacted 事件上自动重新注入
  - 三级技能优先级：project > personal > superpowers
  - 项目本地技能支持（`.opencode/skills/`）
  - 与 Codex 共享核心模块（`lib/skills-core.js`）以实现代码复用
  - 带有适当隔离的自动化测试套件（`tests/opencode/`）
  - 平台特定文档（`docs/README.opencode.md`、`docs/README.codex.md`）

### 变更

- **重构的 Codex 实现**：现在使用共享的 `lib/skills-core.js` ES 模块
  - 消除 Codex 和 OpenCode 之间的代码重复
  - 技能发现和解析的单一真实来源
  - Codex 通过 Node.js 互操作成功加载 ES 模块

- **改进的文档**：重写 README 以清晰解释问题/解决方案
  - 移除了重复部分和冲突信息
  - 添加了完整工作流描述（brainstorm → plan → execute → finish）
  - 简化了平台安装说明
  - 强调技能检查协议而不是自动激活声明

---

## v3.4.1 (2025-10-31)

### 改进

- 优化了 superpowers bootstrap，消除冗余的技能执行。`using-superpowers` 技能内容现在直接在会话上下文中提供，并明确指导仅对其他技能使用 Skill 工具。这减少了开销并防止了智能体会手动执行 `using-superpowers` 的混乱循环，尽管已经从会话开始时获得了内容。

## v3.4.0 (2025-10-30)

### 改进

- 简化了 `brainstorming` 技能，回归原始的对话愿景。移除了带有正式检查清单的重型 6 阶段流程，改为自然对话：一次问一个问题，然后以 200-300 字的部分呈现设计并验证。保留了文档和实施交接功能。

## v3.3.1 (2025-10-28)

### 改进

- 更新了 `brainstorming` 技能，要求在进行提问前进行自主侦察，鼓励推荐驱动的决策，防止智能体将优先级决策推回给人类。
- 按照 Strunk 的"Elements of Style"原则对 `brainstorming` 技能应用了写作清晰度改进（删除了不必要的词汇，将否定形式转换为肯定形式，改进了并列结构）。

### Bug 修复

- 明确了 `writing-skills` 指导，使其指向正确的智能体特定个人技能目录（Claude Code 为 `~/.claude/skills`，Codex 为 `~/.codex/skills`）。

## v3.3.0 (2025-10-28)

### 新功能

**实验性 Codex 支持**
- 添加了统一的 `superpowers-codex` 脚本，包含 bootstrap/use-skill/find-skills 命令
- 跨平台 Node.js 实现（在 Windows、macOS、Linux 上工作）
- 命名空间技能：superpowers 技能为 `superpowers:skill-name`，个人技能为 `skill-name`
- 个人技能在名称匹配时覆盖 superpowers 技能
- 干净的技能显示：显示名称/描述，不包含 raw frontmatter
- 有用的上下文：显示每个技能的支持文件目录
- Codex 的工具映射：TodoWrite→update_plan、subagents→manual fallback 等
- 与最小 AGENTS.md 的 Bootstrap 集成，用于自动启动
- 完整的安装指南和特定于 Codex 的 bootstrap 说明

**与 Claude Code 集成的关键区别：**
- 单一统一脚本，而不是单独的工具
- 用于 Codex 特定等价物的工具替换系统
- 简化的子智能体处理（手动工作而不是委派）
- 更新的术语："Superpowers skills"而不是"Core skills"

### 新增文件
- `.codex/INSTALL.md` - Codex 用户的安装指南
- `.codex/superpowers-bootstrap.md` - 带 Codex 适配的 Bootstrap 指令
- `.codex/superpowers-codex` - 包含所有功能的统一 Node.js 可执行文件

**注意：** Codex 支持是实验性的。集成提供核心 superpowers 功能，但可能需要根据用户反馈进行细化。

## v3.2.3 (2025-10-23)

### 改进

**更新了 using-superpowers 技能以使用 Skill 工具而不是 Read 工具**
- 将技能调用指令从 Read 工具改为 Skill 工具
- 更新了描述："using Read tool" → "using Skill tool"
- 更新了第 3 步："Use the Read tool" → "Use the Skill tool to read and run"
- 更新了理性化列表："Read the current version" → "Run the current version"

Skill 工具是在 Claude Code 中调用技能的正确机制。此更新纠正了 bootstrap 指令，引导智能体使用正确的工具。

### 变更文件
- 更新：`skills/using-superpowers/SKILL.md` - 将工具引用从 Read 改为 Skill

## v3.2.2 (2025-10-21)

### 改进

**增强了 using-superpowers 技能以抵抗智能体理性化**
- 添加了 EXTREMELY-IMPORTANT 块，包含关于强制性技能检查的绝对语言
  - "If even 1% chance a skill applies, you MUST read it"
  - "You do not have a choice. You cannot rationalize your way out."
- 添加了 MANDATORY FIRST RESPONSE PROTOCOL 检查清单
  - 智能体在任何响应之前必须完成的 5 步流程
  - 显式的"responding without this = failure"后果
- 添加了 Common Rationalizations 部分，包含 8 种具体的规避模式
  - "This is just a simple question" → WRONG
  - "I can check files quickly" → WRONG
  - "Let me gather information first" → WRONG
  - 加上在智能体行为中观察到的另外 5 种常见模式

这些更改解决了观察到的智能体行为，即它们尽管有明确的指令也会理性化地绕过技能使用。强硬的措辞和先发制人的反驳旨在使不遵守变得更困难。

### 变更文件
- 更新：`skills/using-superpowers/SKILL.md` - 添加了三个强制执行层以防止跳过技能的理性化行为

## v3.2.1 (2025-10-20)

### 新功能

**代码审查者智能体现在包含在插件中**
- 将 `superpowers:code-reviewer` 智能体添加到插件的 `agents/` 目录
- 智能体提供针对计划和编码标准的系统性代码审查
- 之前要求用户拥有个人智能体配置
- 所有技能引用已更新为使用命名空间 `superpowers:code-reviewer`
- 修复 #55

### 变更文件
- 新增：`agents/code-reviewer.md` - 带审查检查清单和输出格式的智能体定义
- 更新：`skills/requesting-code-review/SKILL.md` - 对 `superpowers:code-reviewer` 的引用
- 更新：`skills/subagent-driven-development/SKILL.md` - 对 `superpowers:code-reviewer` 的引用

## v3.2.0 (2025-10-18)

### 新功能

**brainstorming 工作流中的设计文档**
- 向 brainstorming 技能添加了 Phase 4: Design Documentation
- 设计文档现在在实施前写入 `docs/plans/YYYY-MM-DD-<topic>-design.md`
- 恢复了在技能转换过程中丢失的原始 brainstorming 命令的功能
- 文档在 worktree 设置和实施计划之前编写
- 使用子智能体测试，验证在时间压力下的合规性

### 不兼容变更

**技能引用命名空间标准化**
- 所有内部技能引用现在使用 `superpowers:` 命名空间前缀
- 更新格式：`superpowers:test-driven-development`（之前只是 `test-driven-development`）
- 影响所有 REQUIRED SUB-SKILL、RECOMMENDED SUB-SKILL 和 REQUIRED BACKGROUND 引用
- 与技能使用 Skill 工具调用的方式对齐
- 更新文件：brainstorming、executing-plans、subagent-driven-development、systematic-debugging、testing-skills-with-subagents、writing-plans、writing-skills

### 改进

**设计与实施计划命名**
- 设计文档使用 `-design.md` 后缀以防止文件名冲突
- 实施计划继续使用现有的 `YYYY-MM-DD-<feature-name>.md` 格式
- 两者都存储在 `docs/plans/` 目录中，具有清晰的命名区分

## v3.1.1 (2025-10-17)

### Bug 修复

- **修复了 README 中的命令语法** (#44) - 更新了所有命令引用以使用正确的命名空间语法（`/superpowers:brainstorm` 而不是 `/brainstorm`）。插件提供的命令由 Claude Code 自动命名空间化，以避免插件间的冲突。

## v3.1.0 (2025-10-17)

### 不兼容变更

**技能名称标准化为小写**
- 所有技能 frontmatter `name:` 字段现在使用小写 kebab-case，匹配目录名
- 示例：`brainstorming`、`test-driven-development`、`using-git-worktrees`
- 所有技能公告和交叉引用已更新为小写格式
- 这确保目录名、frontmatter 和文档之间的命名一致

### 新功能

**增强的 brainstorming 技能**
- 添加了 Quick Reference 表格，显示阶段、活动和工具使用
- 添加了可复制的用于跟踪进度的工作流检查清单
- 添加了决定何时重新访问早期阶段的决策流程图
- 添加了全面的 AskUserQuestion 工具指导，包含具体示例
- 添加了"Question Patterns"部分，解释何时使用结构化与开放式问题
- 将 Key Principles 重构为可扫描的表格

**Anthropic 最佳实践集成**
- 添加了 `skills/writing-skills/anthropic-best-practices.md` - 官方 Anthropic 技能创作指南
- 在 writing-skills SKILL.md 中引用，提供全面指导
- 提供渐进式披露、工作流和评估的模式

### 改进

**技能交叉引用清晰度**
- 所有技能引用现在使用显式要求标记：
  - `**REQUIRED BACKGROUND:**` - 你必须理解的先决条件
  - `**REQUIRED SUB-SKILL:**` - 必须在工作流中使用的技能
  - `**Complementary skills:**` - 可选但有用的相关技能
- 移除了旧路径格式（`skills/collaboration/X` → 只是 `X`）
- 更新了 Integration 部分，包含分类关系（Required vs Complementary）
- 使用最佳实践更新了交叉引用文档

**与 Anthropic 最佳实践对齐**
- 修复了描述语法和语调（完全第三人称）
- 添加了 Quick Reference 表格以便扫描
- 添加了 Claude 可以复制和跟踪的工作流检查清单
- 在非显而易见的决策点适当使用流程图
- 改进了可扫描的表格格式
- 所有技能都在 500 行建议以下

### Bug 修复

- **重新添加了缺失的命令重定向** - 恢复了在 v3.0 迁移中意外删除的 `commands/brainstorm.md` 和 `commands/write-plan.md`
- 修复了 `defense-in-depth` 名称不匹配（是 `Defense-in-Depth-Validation`）
- 修复了 `receiving-code-review` 名称不匹配（是 `Code-Review-Reception`）
- 修复了 `commands/brainstorm.md` 引用正确的技能名称
- 移除了对不存在相关技能的引用

### 文档

**writing-skills 改进**
- 使用显式要求标记更新了交叉引用指导
- 添加了对 Anthropic 官方最佳实践的引用
- 改进了显示正确技能引用格式的示例

## v3.0.1 (2025-10-16)

### 变更

我们现在使用 Anthropic 的第一方技能系统！

## v2.0.2 (2025-10-12)

### Bug 修复

- **修复了本地技能仓库领先于上游时的错误警告** - 初始化脚本在本地仓库有领先于上游的提交时错误地警告"New skills available from upstream"。逻辑现在正确区分三种 git 状态：本地落后（应更新）、本地领先（无警告）和分叉（应警告）。

## v2.0.1 (2025-10-12)

### Bug 修复

- **修复了插件上下文中的 session-start hook 执行** (#8, PR #9) - Hook 静默失败，报"Plugin hook error"，阻止技能上下文加载。修复方法：
  - 当 BASH_SOURCE 在 Claude Code 的执行上下文中未绑定时，使用 `${BASH_SOURCE[0]:-$0}` fallback
  - 添加 `|| true` 以在过滤状态标志时优雅地处理空的 grep 结果

---

# Superpowers v2.0.0 发布说明

## 概述

Superpowers v2.0 通过一次重大架构转变，使技能更易于访问、维护和社区驱动。

核心变化是**技能仓库分离**：所有技能、脚本和文档已从插件移至一个专用仓库（[obra/superpowers-skills](https://github.com/obra/superpowers-skills)）。这将 superpowers 从单体插件转变为一个轻量级 shim，管理技能仓库的本地 clone。技能在会话启动时自动更新。用户 fork 并通过标准 git 工作流贡献改进。技能库独立于插件进行版本管理。

除了基础设施之外，本次发布还新增了九个专注于问题解决、研究和架构的技能。我们使用命令式语调和更清晰的结构重写了核心的 **using-skills** 文档，使 Claude 更容易理解何时以及如何使用技能。**find-skills** 现在输出可直接粘贴到 Read 工具的路径，消除了技能发现工作流中的摩擦。

用户体验无缝操作：插件自动处理 clone、fork 和更新。贡献者发现新架构使改进和分享技能变得简单。此发布为技能作为社区资源快速发展奠定了基础。

## 不兼容变更

### 技能仓库分离

**最大的变化：** 技能不再存在于插件中。它们已移至 [obra/superpowers-skills](https://github.com/obra/superpowers-skills) 的独立仓库。

**这对你意味着什么：**

- **首次安装：** 插件自动将技能 clone 到 `~/.config/superpowers/skills/`
- **Fork：** 设置过程中，系统会提供 fork 技能仓库的选项（如果安装了 `gh`）
- **更新：** 技能在会话启动时自动更新（在可能时 fast-forward）
- **贡献：** 在分支上工作、本地提交、向上游提交 PR
- **不再有 shadowing：** 旧的二级系统（personal/core）替换为单仓库分支工作流

**迁移：**

如果你已有安装：
1. 你的旧 `~/.config/superpowers/.git` 将备份到 `~/.config/superpowers/.git.bak`
2. 旧技能将备份到 `~/.config/superpowers/skills.bak`
3. obra/superpowers-skills 的全新 clone 将在 `~/.config/superpowers/skills/` 处创建

### 移除的功能

- **个人 superpowers overlay 系统** - 替换为 git 分支工作流
- **setup-personal-superpowers hook** - 替换为 initialize-skills.sh

## 新功能

### 技能仓库基础设施

**自动 Clone 和设置** (`lib/initialize-skills.sh`)
- 首次运行时 Clone obra/superpowers-skills
- 如果安装了 GitHub CLI，提供创建 fork 的选项
- 正确设置 upstream/origin 远程
- 处理来自旧安装的迁移

**自动更新**
- 每次会话启动时从跟踪的远程 fetch
- 在可能时 auto-merge 使用 fast-forward
- 需要手动同步时通知（分支分叉）
- 使用 pulling-updates-from-skills-repository 技能进行手动同步

### 新技能

**问题解决技能** (`skills/problem-solving/`)
- **collision-zone-thinking** - 强制不相关概念碰撞以获得涌现洞察
- **inversion-exercise** - 翻转假设以揭示隐藏约束
- **meta-pattern-recognition** - 跨领域发现通用原则
- **scale-game** - 在极端条件下测试以揭示基本真理
- **simplification-cascades** - 找到消除多个组件的洞察
- **when-stuck** - 分派到正确的问题解决技术

**研究技能** (`skills/research/`)
- **tracing-knowledge-lineages** - 理解思想如何随时间演变

**架构技能** (`skills/architecture/`)
- **preserving-productive-tensions** - 保持多种有效方法而不是强制过早解决

### 技能改进

**using-skills（原名 getting-started）**
- 从 getting-started 重命名为 using-skills
- 使用命令式语气的完全重写（v4.0.0）
- 关键规则前置
- 为所有工作流添加了"Why"解释
- 在引用中始终包含 /SKILL.md 后缀
- 刚性规则和灵活模式之间更清晰的区分

**writing-skills**
- 交叉引用指导从 using-skills 移入
- 添加了 token 效率部分（字数目标）
- 改进了 CSO（Claude Search Optimization）指导

**sharing-skills**
- 为新分支和 PR 工作流更新（v2.0.0）
- 移除了 personal/core split 引用

**pulling-updates-from-skills-repository**（新增）
- 完整的与上游同步的工作流
- 替换旧的"updating-skills"技能

### 工具改进

**find-skills**
- 现在输出带有 /SKILL.md 后缀的完整路径
- 使路径可直接用于 Read 工具
- 更新了帮助文本

**skill-run**
- 从 scripts/ 移至 skills/using-skills/
- 改进了文档

### 插件基础设施

**Session Start Hook**
- 现在从技能仓库位置加载
- 在会话启动时显示完整技能列表
- 打印技能位置信息
- 显示更新状态（updated successfully / behind upstream）
- 将"skills behind"警告移到输出末尾

**环境变量**
- `SUPERPOWERS_SKILLS_ROOT` 设置为 `~/.config/superpowers/skills`
- 在所有路径中一致使用

## Bug 修复

- 修复了 fork 时的重复 upstream 远程添加
- 修复了 find-skills 输出中的双重"skills/"前缀
- 从 session-start 中移除了过时的 setup-personal-superpowers 调用
- 修复了所有 hooks 和命令中的路径引用

## 文档

### README
- 为新技能仓库架构更新
- 指向 superpowers-skills 仓库的醒目链接
- 更新了自动更新描述
- 修复了技能名称和引用
- 更新了 Meta 技能列表

### 测试文档
- 添加了全面的测试检查清单（`docs/TESTING-CHECKLIST.md`）
- 为测试创建了本地市场配置
- 文档化了手动测试场景

## 技术细节

### 文件变更

**新增：**
- `lib/initialize-skills.sh` - 技能仓库初始化和自动更新
- `docs/TESTING-CHECKLIST.md` - 手动测试场景
- `.claude-plugin/marketplace.json` - 本地测试配置

**移除：**
- `skills/` 目录（82 个文件） - 现在在 obra/superpowers-skills
- `scripts/` 目录 - 现在在 obra/superpowers-skills/skills/using-skills/
- `hooks/setup-personal-superpowers.sh` - 过时

**修改：**
- `hooks/session-start.sh` - 从 ~/.config/superpowers/skills 使用技能
- `commands/brainstorm.md` - 更新路径为 SUPERPOWERS_SKILLS_ROOT
- `commands/write-plan.md` - 更新路径为 SUPERPOWERS_SKILLS_ROOT
- `commands/execute-plan.md` - 更新路径为 SUPERPOWERS_SKILLS_ROOT
- `README.md` - 为新架构完全重写

### 提交历史

本次发布包括：
- 20+ 个用于技能仓库分离的提交
- PR #1：受 Amplifier 启发的问题解决和研究技能
- PR #2：个人 superpowers overlay 系统（后来被替换）
- 多项技能细化和文档改进

## 升级说明

### 全新安装

```bash
# 在 Claude Code 中
/plugin marketplace add obra/superpowers-marketplace
/plugin install superpowers@superpowers-marketplace
```

插件自动处理一切。

### 从 v1.x 升级

1. **备份你的个人技能**（如果你有的话）：
   ```bash
   cp -r ~/.config/superpowers/skills ~/superpowers-skills-backup
   ```

2. **更新插件：**
   ```bash
   /plugin update superpowers
   ```

3. **在下次会话启动时：**
   - 旧安装将自动备份
   - 新技能仓库将被 clone
   - 如果你有 GitHub CLI，系统会提供 fork 的选项

4. **迁移个人技能**（如果你有的话）：
   - 在你的本地技能仓库中创建分支
   - 从备份复制你的个人技能
   - 提交并推送到你的 fork
   - 考虑通过 PR 回馈

## 下一步

### 对于用户

- 探索新的问题解决技能
- 尝试基于分支的技能改进工作流
- 将技能回馈给社区

### 对于贡献者

- 技能仓库现在位于 https://github.com/obra/superpowers-skills
- Fork → Branch → PR 工作流
- 参见 skills/meta/writing-skills/SKILL.md 了解 TDD 文档方法

## 已知问题

目前没有。

## 致谢

- 问题解决技能受 Amplifier 模式启发
- 社区贡献和反馈
- 对技能有效性的广泛测试和迭代

---

**完整变更日志：** https://github.com/obra/superpowers/compare/dd013f6...main
**技能仓库：** https://github.com/obra/superpowers-skills
**Issues：** https://github.com/obra/superpowers/issues
