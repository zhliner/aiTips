# AITips - 个人 AI 小工具集

存放一些个人使用的 AI 编码用提示词（*.md），如 commands、系统级 AGENTS.md 等。
以及其它一些零碎物件。


## 工作流（Human & AI）

1. 构想（Conception）。项目发起，人类主导。
2. 决策（Decision）。AI 提供建议，人类主导。
3. 提案（Proposal/Spec）。技术提案、规格。明确边界，AI 生成，人类审阅。
4. 实现（Implementation Plan）。AI 根据提案创建。
    - 设计，编码级。
    - 任务清单，实现步骤。


## 内容

### 系统级提示

- AGENTS.md - 系统级常驻提示，主要补充本地化规则。
- AGENTS@docs.md - 项目 docs/ 目录下的基础提示，定义项目 4 级规则（构想/决策/提案/实现）。


### 技能（Skills）

[root]: ~/.config/opencode/skills/

agent-skills/
github.com/addyosmani/agent-skills 的 24 个技能，有谷歌资深工程师编写。
- references -> /Users/dev/aiTips/addyosmani/references
- scripts -> /Users/dev/aiTips/addyosmani/scripts
- skills -> /Users/dev/aiTips/addyosmani/skills

agent-skills/skills/
- api-and-interface-design
- browser-testing-with-devtools
- ci-cd-and-automation
- code-review-and-quality
- code-simplification
- context-engineering
- debugging-and-error-recovery
- deprecation-and-migration
- documentation-and-adrs
- doubt-driven-development
- frontend-ui-engineering
- git-workflow-and-versioning
- idea-refine
- incremental-implementation
- interview-me
- observability-and-instrumentation
- performance-optimization
- planning-and-task-breakdown
- security-and-hardening
- shipping-and-launch
- source-driven-development
- spec-driven-development
- test-driven-development
- using-agent-skills

superpowers/skills/
github.com/obra/superpowers 手动集成其中的 9 个技能，主要修改强硬措辞和 `superpowers:` 前缀等。
- brainstorming -> /Users/dev/aiTips/superpowers/skills/brainstorming
- dispatching-parallel-agents -> /Users/dev/aiTips/superpowers/skills/dispatching-parallel-agents
- executing-plans -> /Users/dev/aiTips/superpowers/skills/executing-plans
- finishing-a-development-branch -> /Users/dev/aiTips/superpowers/skills/finishing-a-development-branch
- requesting-code-review -> /Users/dev/aiTips/superpowers/skills/requesting-code-review
- subagent-driven-development -> /Users/dev/aiTips/superpowers/skills/subagent-driven-development
- using-git-worktrees -> /Users/dev/aiTips/superpowers/skills/using-git-worktrees
- verification-before-completion -> /Users/dev/aiTips/superpowers/skills/verification-before-completion
- writing-plans -> /Users/dev/aiTips/superpowers/skills/writing-plans

anthropics/skills/
github.com/anthropics/skills 集成该经典的技能（行业默认标准）
- skill-creator -> /Users/dev/aiTips/anthropics/skills/skill-creator


### 代理（Agents）

实际使用的代理仅包含：
- agents/ 下自定义的 7 个 Agents（中文版）。
- addyosmani/agents/ 下的 4 个 Agents（英文原版）。