# Agent Personas（智能体角色）

专家角色，每个角色承担单一职责、提供单一视角。每个角色以 Markdown 文件形式存在，作为系统提示词被你的运行环境（Claude Code、Cursor、Copilot 等）加载。

| 角色 | 职责 | 最佳适用场景 |
|------|------|-------------|
| [code-reviewer](../agents/code-reviewer.md) | 高级工程师 | 合并前的五维度代码审查 |
| [security-auditor](../agents/security-auditor.md) | 安全工程师 | 漏洞检测、OWASP 风格审计 |
| [test-engineer](../agents/test-engineer.md) | QA 工程师 | 测试策略、覆盖率分析、Prove-It 模式 |
| [web-performance-auditor](../agents/web-performance-auditor.md) | Web 性能工程师 | Core Web Vitals 审计、加载/渲染/网络分析 |

## 角色与技能和命令的关系（How personas relate to skills and commands）

三个层次，各司其职：

| 层次 | 定义 | 示例 | 组合角色 |
|------|------|------|---------|
| **技能（Skill）** | 包含步骤和退出条件的工作流 | `code-review-and-quality` | *怎么做* — 在角色或命令内部调用 |
| **角色（Persona）** | 带有特定视角和输出格式的角色 | `code-reviewer` | *谁来做* — 采用特定视角，生成报告 |
| **命令（Command）** | 面向用户的入口 | `/review`、`/ship` | *何时做* — 组合角色和技能 |

用户（或斜杠命令）是编排者。**角色不会调用其他角色。** 技能是角色工作流中的必经节点。

## 何时使用哪种方式（When to use each）

### 直接调用角色（Direct persona invocation）
当你需要对当前变更获取单一视角，且用户在循环中时，选择此方式。

- "审查这个 PR" → 直接调用 `code-reviewer`
- "`auth.ts` 中有安全问题吗？" → 直接调用 `security-auditor`
- "结账流程缺少哪些测试？" → 直接调用 `test-engineer`
- "审计产品页面的 Core Web Vitals" → 直接调用 `web-performance-auditor`

### 斜杠命令（单一角色驱动）（Slash command - single persona behind it）
当你有一个可重复的工作流，不想每次都重新解释时，选择此方式。

- `/review` → 封装 `code-reviewer` 和项目的审查技能
- `/test` → 封装 `test-engineer` 和 TDD 技能
- `/webperf` → 封装 `web-performance-auditor`，用于 Web 应用的性能审计

### 斜杠命令（编排器 — 扇出模式）（Slash command - orchestrator fan-out）
仅当**独立的**调查可以并行运行，并由单个智能体合并报告时，选择此方式。

- `/ship` → 并行扇出到 `code-reviewer` + `security-auditor` + `test-engineer`，然后将报告综合为通过/不通过决策

这是本仓库认可的唯一编排模式。完整的模式目录和反模式请参阅 [references/orchestration-patterns.md](../references/orchestration-patterns.md)。

## 决策矩阵（Decision matrix）

```
该工作是否是对单一产物的单一视角分析？
├── 是 → 直接调用角色
└── 否  → 子任务是否相互独立（无共享可变状态，无顺序依赖）？
         ├── 是 → 使用并行扇出的斜杠命令（如 /ship）
         └── 否  → 用户按顺序执行斜杠命令（/spec → /plan → /build → /test → /review）
```

## 实例：有效的编排（Worked example: valid orchestration）

`/ship` 是本仓库中规范的扇出编排器：

```
/ship
  ├── （并行）code-reviewer    → 审查报告
  ├── （并行）security-auditor → 审计报告
  └── （并行）test-engineer    → 覆盖率报告
                  ↓
        合并阶段（主智能体）
                  ↓
        通过/不通过决策 + 回滚计划
```

为什么这样可行：
- 每个子智能体操作同一个 diff，但产出**不同视角**的分析
- 它们之间没有依赖 → 真正的并行，实际节省时间
- 每个子智能体在独立的上下文窗口中运行 → 主会话保持整洁
- 合并步骤较小且受益于完整上下文，因此留在主智能体中

## 实例：无效的编排（请勿构建）（Worked example: invalid orchestration）

一个 `meta-orchestrator` 角色，其职责是"决定调用哪个其他角色"：

```
/work-on-pr → meta-orchestrator
                   ↓ （决定"这需要一次审查"）
               code-reviewer
                   ↓ （返回结果）
               meta-orchestrator（转述结果）
                   ↓
               用户
```

为什么这样不行：
- 纯粹的路由层，没有领域价值
- 增加了两次转述跳转 → 信息丢失 + 2 倍 token 消耗
- 用户已经知道自己想要审查；让他们直接调用 `/review`
- 重复了斜杠命令和 `AGENTS.md` 意图映射已经做的工作

## 角色规则（Rules for personas）

1. 一个角色是单一职责配单一输出格式。如果你发现需要添加第二个职责，请创建第二个角色。
2. **角色不调用其他角色。** 组合是斜杠命令或用户的工作。在 Claude Code 中这也是硬性平台约束 — *"子智能体不能生成其他子智能体"* — 因此该规则由平台强制执行。
3. 角色可以调用技能（即*怎么做*）。
4. 每个角色文件以"组合（Composition）"块结尾，说明其适用场景。

## Claude Code 互操作（Claude Code interop）

本仓库中的角色设计为可直接作为 Claude Code 子智能体和 Agent Teams 队友使用，无需修改：

- **作为子智能体：** 启用此插件后自动发现（无需路径配置）。使用 Agent 工具时指定 `subagent_type: code-reviewer`（或 `security-auditor`、`test-engineer`）。`/ship` 是规范示例。
- **作为 Agent Teams 队友**（实验性功能，需要 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`）：生成队友时引用相同的角色名称。角色内容会被**追加到**队友的系统提示词中作为额外指令（而非替换），因此你的角色文本会叠加在 lead 安装的团队协调指令（SendMessage、task-list 工具等）之上。

子智能体仅将结果报告回主智能体。Agent Teams 允许队友之间直接通信。当报告足够时使用子智能体；当子智能体需要相互质疑发现时使用 Agent Teams（如竞争假设调试）。完整映射请参阅 [references/orchestration-patterns.md](../references/orchestration-patterns.md)。

插件智能体不支持 `hooks`、`mcpServers` 或 `permissionMode` frontmatter — 这些字段会被静默忽略。在此处编写新角色时请避免依赖它们。

## 添加新角色（Adding a new persona）

1. 创建 `agents/<role>.md`，使用与现有角色相同的 frontmatter 格式。
2. 定义角色、范围、输出格式和规则。
3. 在底部添加**组合（Composition）**块（直接调用时机 / 通过什么调用 / 不要从其他角色调用）。
4. 将角色添加到本文件顶部的表格中。
5. 如果该角色启用了新的编排模式，请在 `references/orchestration-patterns.md` 中记录，而不是在角色文件本身中发明模式。
