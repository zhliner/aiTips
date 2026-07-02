# Orchestration Patterns（编排模式）

本仓库认可的 Agent 编排模式参考目录，以及应避免的反模式。在添加新的协调多个 Persona 的 slash command 之前，或在引入新的"包装"现有 Persona 的 Persona 之前，请先阅读本文。

核心规则：**用户（或 slash command）是编排者。Persona 不调用其他 Persona。** Skill 是 Persona 工作流内部的强制步骤。

---

## Endorsed Patterns（认可的编排模式）

### 1. Direct Invocation（直接调用——无编排）

单一 Persona，单一视角，单一产出物。默认选项，也是最经济的选项。

```
user → code-reviewer → report → user
```

**适用场景：** 工作是对一个产出物的一个视角，且能用一句话描述。

**示例：**
- "Review this PR" → `code-reviewer`
- "Find security issues in `auth.ts`" → `security-auditor`
- "What tests are missing for the checkout flow?" → `test-engineer`

**成本：** 一次往返。应始终将其他编排模式与此基线对比。

---

### 2. Single-Persona Slash Command（单一 Persona 的 Slash Command）

一个 slash command 包装一个 Persona 并附上项目的 Skill。免去用户每次都重新解释工作流。

```
/review → code-reviewer（附带 code-review-and-quality skill）→ report
```

**适用场景：** 同一单个 Persona 调用以相同设置反复出现。

**本仓库中的示例：** `/review`、`/test`、`/code-simplify`。

**成本：** 与直接调用相同。Slash command 只是一个保存好的 prompt。

**反信号：** 如果 slash command 的主体大部分是"决定调用哪个 Persona"，则删除它，让用户直接调用 Persona。

---

### 3. Parallel Fan-Out with Merge（并行扇出与合并）

多个 Persona 同时对相同输入进行操作，各自生成独立报告。合并步骤（在主 Agent 上下文中）将它们综合为一个决策。

```
                    ┌─→ code-reviewer    ─┐
/ship → fan out  ───┼─→ security-auditor ─┤→ merge → go/no-go + rollback
                    └─→ test-engineer    ─┘
```

**适用场景：**
- 子任务之间真正独立（无共享可变状态，无顺序依赖）
- 每个 Subagent 受益于自己的上下文窗口
- 合并步骤足够小，能留在主上下文中
- 墙上时钟延迟很重要

**本仓库中的示例：** `/ship`。

**成本：** N 个并行 Subagent 上下文 + 一次合并回合。比直接调用更高，但墙上时钟更快，且由于每个 Subagent 专注于单一视角，产生的报告质量更好。

**采用此模式前的验证清单：**
- [ ] 能否同时运行所有 Subagent 而不存在顺序问题？
- [ ] 每个 Persona 是否产生不同*种类*的发现，而不仅仅是从不同角度得出相同的发现？
- [ ] 合并步骤能否装进主 Agent 剩余上下文中？
- [ ] 用户的等待时间是否足够长，使得并行效果真正可感知？

如果任何一个答案为"否"，则退回到直接调用或单一 Persona command。

---

### 4. Sequential Pipeline as User-Driven Slash Commands（作为用户驱动的 Slash Command 的顺序流水线）

用户按定义好的顺序运行 slash command，在它们之间传递上下文（或提交历史）。没有编排 Agent——用户就是编排者。

```
user runs:  /spec  →  /plan  →  /build  →  /test  →  /review  →  /ship
```

**适用场景：** 工作流有依赖关系（每一步需要前一步的输出），且步骤之间的人类判断能增加价值。

**本仓库中的示例：** 整个 DEFINE → PLAN → BUILD → VERIFY → REVIEW → SHIP 生命周期。

**成本：** 每个步骤一个 Subagent 上下文。编排层零成本，因为没有编排 Agent。

**为什么不自动化：** LLM "生命周期编排器" 会（a）因为需要为传递而总结，导致步骤之间丢失细微差别，（b）跳过能早期发现方向错误的人类检查点，（c）通过转述回合使 Token 成本翻倍。

---

### 5. Research Isolation（研究隔离——上下文保护）

当任务需要阅读大量材料而可能污染主上下文时，派生一个 Research Subagent，仅返回摘要。

```
main agent → research sub-agent（读取 50 个文件）→ digest → main agent continues
```

**适用场景：**
- 主会话需要专注于下游任务
- 调查结果远小于它消耗的输入
- 决策质量受益于主 Agent 事后有充分思考空间

**示例：** "Find every call site of this deprecated API across the monorepo"、"Summarize what these 30 ADRs say about caching."

**成本：** 一个隔离的 Subagent 上下文。相比在主上下文中加载数百个文件的替代方案，这是值得的。

**在 Claude Code 上，使用内置的 `Explore` Subagent**，而非定义自定义 Research Persona。`Explore` 运行在 Haiku 上，被禁止 write/edit 工具，专为此模式设计。仅在 `Explore` 不适合时定义自定义 Research Subagent（例如你需要模型无法自行推断的领域专用 system prompt）。

---

## Claude Code Compatibility（Claude Code 兼容性）

本目录与工具无关，但大多数读者将在 Claude Code 上运行。以下说明了每种模式如何映射到 Claude Code 的原语——以及平台在哪些地方帮我们强制执行了规则。

### Persona 的存放位置

Plugin Subagent 存放在 Plugin 根目录下的 `agents/` 中。本仓库是一个 Plugin（`.claude-plugin/plugin.json`），因此 `agents/code-reviewer.md`、`agents/security-auditor.md` 和 `agents/test-engineer.md` 在启用 Plugin 时会被自动发现。无需路径配置。

### Subagents vs. Agent Teams

Claude Code 有两种并行原语。模式 3（并行扇出与合并）映射到 **Subagents**。如果需要互相通信的团队成员，则使用 **Agent Teams**。

| | Subagents | Agent Teams |
|--|-----------|-------------|
| 协调方式 | 主 Agent 扇出，Subagent 仅回报 | 团队成员互相发消息，共享任务列表 |
| 上下文 | 每个 Subagent 有独立的上下文窗口 | 每个队友有独立的上下文窗口 |
| 适用场景 | 产生报告的独立任务 | 需要讨论的协作工作 |
| 状态 | 稳定 | 实验性——需要 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` |
| 成本 | 较低 | 较高——每个队友是独立的 Claude 实例 |

**本仓库中的 Persona 在两种模式下都能工作。** 当作为 Subagent 派生时（例如由 `/ship` 派生），它们将发现报告给主会话。当作为 Teammate 派生时（`Spawn a teammate using the security-auditor agent type…`），它们可以直接互相质疑对方的发现。Persona 定义是相同的；只有派生上下文不同。

一个细微之处：Persona 中的 `skills` 和 `mcpServers` frontmatter 字段在作为 Subagent 运行时会生效，但在作为 Teammate 运行时**会被忽略**——Teammate 从你的项目和用户设置中加载 Skill 和 MCP Server，与常规会话相同。如果某个 Persona 依赖特定的 Skill 或 MCP Server 被加载，请在会话级别配置它们，以便在两种模式下都可用。

### Platform-Enforced Rules（平台强制规则）

本目录中有两条规则不仅仅是约定——Claude Code 强制执行它们：

- **"Subagent 不能派生其他 Subagent"**（文档原文）。反模式 B（Persona 调用 Persona）和反模式 D（深层 Persona 树）在 Claude Code 上从结构上就不可能出现。
- **"不能嵌套 Team"**——Teammate 不能派生出自己的 Team。相同的反模式在 Team 级别被阻止。

这意味着你可以采用本目录中的模式，而不必担心贡献者意外构建出反模式。它们根本加载不了。

### Built-in Subagents（内置 Subagent 须知）

在定义自定义 Subagent 之前，检查以下内置 Subagent 是否已涵盖该角色：

| 内置 Subagent | 用途 |
|----------|---------|
| `Explore` | 只读的代码库搜索与分析。用于模式 5（研究隔离）。 |
| `Plan` | Plan 模式下的只读研究。 |
| `general-purpose` | 需要探索和修改的多步骤任务。 |

不要重新定义这些。将你的专家级 Persona（code-reviewer、security-auditor、test-engineer）建立在它们之上。

### Plugin Agent 的 Frontmatter 限制

Plugin Subagent **不**支持 `hooks`、`mcpServers` 或 `permissionMode` frontmatter 字段——这些将被静默忽略。如果未来某个 Persona 需要其中任何一个，用户必须将文件复制到 `.claude/agents/` 或 `~/.claude/agents/` 中。

在 Plugin Agent 中**有效**的字段有：`name`、`description`、`tools`、`disallowedTools`、`model`、`maxTurns`、`skills`、`memory`、`background`、`effort`、`isolation`、`color`、`initialPrompt`。如果你想优化成本，可以按 Persona 使用 `model`（例如 `test-engineer` 覆盖度扫描用 Haiku，`code-reviewer` 用 Sonnet，`security-auditor` 用 Opus）。

### 并行派生多个 Subagent

在 Claude Code 中，并行扇出（模式 3）需要在**单个 Assistant 回合中发出多个 Agent 工具调用**。顺序回合会串行执行。`/ship` 对此有明确说明。任何新的编排器 Command 也应这样做。

---

## Worked Example: Agent Teams for Competing-Hypothesis Debugging（实践示例：竞争假设调试的 Agent Teams）

此示例展示了何时使用 **Agent Teams** 而非 `/ship` 的 Subagent 扇出。这两种模式从远处看很相似——都派生相同的三个 Persona——但价值来源不同。

### 场景

> *结账偶尔会挂起约 30 秒才能完成。大约每 50 个会话发生一次。日志中没有错误。从上周的发布开始出现。*

可能的根本原因（互相排斥，但都符合症状）：

1. 新支付确认流程中的竞态条件
2. 一个偶尔会 fall through 到慢速同步网络调用的身份验证检查
3. 某个查询上缺少索引，随购物车大小扩展
4. 一个不稳定的第三方 API，其 SDK 在超时前静默重试

单个 Agent 会选定第一个合理的理论并停止调查。`/ship` 风格的 Subagent 扇出会让每个 Persona 独立报告——但它们的报告永远不会相遇，因此无法排除错误的理论。

这正是 Agent Teams 文档所描述的情况：*"多个独立调查者积极尝试互相否定，存活下来的理论更有可能是真正的根本原因。"*

### 为什么这*不是* `/ship` 的工作

| | `/ship`（Subagents） | Agent Teams |
|--|--------------------|-------------|
| Sub-agent 看到的是 | 同一个 diff，不同的视角 | 共享的任务列表，彼此的消息 |
| 产出是 | 三份独立报告 → 一次合并 | 对抗性辩论 → 共识根本原因 |
| 适用于 | 对已知制品做出结论 | 在多个假设中*寻找*制品 |

`/ship` 是一种判决；Agent Teams 是一种调查。

### Setup（一次性，每个环境）

Agent Teams 是实验性的。在 `~/.claude/settings.json` 中：

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

需要 Claude Code v2.1.32 或更高版本。本仓库中的 Persona 会被自动识别——无需手工编写 Team 配置文件。

### 触发 Prompt

在主导会话中，用自然语言输入：

```
Users report checkout hangs for ~30 seconds intermittently after last
week's release. No errors in logs.

Create an agent team to debug this with competing hypotheses. Spawn
three teammates using the existing agent types:

  - code-reviewer  — investigate race conditions and blocking calls
                     in the checkout code path
  - security-auditor — investigate auth checks, session handling,
                       and any synchronous network calls added recently
  - test-engineer  — propose tests that would distinguish between the
                     hypotheses and check coverage gaps in checkout

Have them message each other directly to challenge each other's
theories. Update findings as consensus emerges. Only converge when
two teammates agree they can disprove the others'.
```

主导会话派生三个引用现有 Persona 名称的 Teammate。Persona 正文将**追加**到每个 Teammate 的 system prompt 中作为附加指令（在主导会话安装的团队协调指令之上）；上述触发 Prompt 成为它们的任务。

### What Happens（发生了什么）

1. 每个 Teammate 在其自己的上下文窗口中运行，从自己的视角探索代码库。
2. Teammate 使用 `message` 直接向彼此发送发现。主导会话无需中转。
3. 共享的任务列表显示谁在调查什么——随时可通过 `Ctrl+T`（in-process 模式）或 tmux 窗格（split 模式）查看。
4. 当 `code-reviewer` 发现一个本应是顺序执行的 `Promise.all` 时，它向 `security-auditor` 发消息确认 auth 调用不是竞态条件的一部分。`security-auditor` 进行检查并回复——要么确认竞态条件是真问题，要么提出反证。
5. `test-engineer` 为当前领先的理论提议一个聚焦的集成测试，团队用来验证后再宣布达成共识。
6. 主导会话综合收敛后的发现并呈现给你。

你可以随时通过 `Shift+Down` 循环切换到任何 Teammate 并输入——对于纠正走错路的调查员很有用。

### 何时清理

当调查找到根本原因时，告诉主导会话：

```
Clean up the team
```

始终通过主导会话清理，而不是通过 Teammate（根据文档：Teammate 缺少完整团队上下文来执行清理）。

### 成本预期

三个 Sonnet Teammate 运行约 10–15 分钟的调查，成本明显比 `/ship` 将同样三个 Persona 作为 Subagent 派生更高。但理由是*结论的质量*——对于错误修复代价高昂的生产调试，额外的 Token 是划算的。对于常规 PR Review，坚持使用 `/ship`。

### 此场景下的反模式

**不要**将此重建为扇出 Subagent 的 `/debug` slash command。Subagent 无法互相发消息——你将失去使该模式生效的对抗性辩论。如果某个工作流反复出现，将上述触发 Prompt 作为代码片段记录下来，而不是将其包装在误用 Subagent 的 slash command 中。

### 何时*不*使用 Agent Teams

- 对已知 diff 做出面向生产的结论 → 使用 `/ship`（Subagents）。
- 对一个制品的单一专家视角 → 直接 Persona 调用。
- 顺序生命周期（spec → plan → build）→ 用户驱动的 slash command（模式 4）。
- 读密集型研究且摘要较小 → 内置 `Explore` Subagent。

仅在 Teammate **需要**互相质疑才能产生正确答案时，才使用 Agent Teams。

---

## Anti-Patterns（反模式）

### A. Router Persona（路由 Persona——"元编排器"）

一个任务是决定调用哪个其他 Persona 的 Persona。

```
/work → router-persona → "this needs a review" → code-reviewer → router（转述）→ user
```

**为什么失败：**
- 纯路由层，无领域价值
- 增加两次转述跳跃 → 信息丢失 + 大约 2 倍 Token 成本
- 用户本就知道他们想要 review；他们本可以直接调用 `/review`
- 重复了 slash command 和 `AGENTS.md` 中意图映射已做的工作

**替代方案：** 添加或改进 slash command。在 `AGENTS.md` 中记录意图 → Command 映射。

---

### B. Persona That Calls Another Persona（Persona 调用另一个 Persona）

一个 `code-reviewer` 在看到 auth 代码时内部调用 `security-auditor`。

**为什么失败：**
- Persona 被设计为产生单一视角；链接它们违背了这一点
- 调用方 Persona 传递的摘要会丢失被调用 Persona 需要的上下文
- 失败模式倍增（谁的输出格式获胜？谁的规则适用？）
- 对用户隐藏成本

**替代方案：** 让调用方 Persona 在其报告中*建议*进行后续审计。用户或 slash command 运行第二遍。

---

### C. Sequential Orchestrator That Paraphrases（转述的顺序编排器）

一个 Agent 代表用户依次调用 `/spec`、然后 `/plan`、然后 `/build` 等。

**为什么失败：**
- 丢失了能捕获方向错误工作的人类检查点
- 每次传递都会总结上下文——长流水线中的累积漂移
- Token 成本翻倍：编排器回合 + 每个步骤的 Subagent 回合
- 在判断最重要的节点处剥夺了用户权代理

**替代方案：** 保持用户作为编排者。在 `README.md` 中记录推荐的顺序，让用户来调用。

---

### D. Deep Persona Trees（深层 Persona 树）

`/ship` 调用 `pre-ship-coordinator`，后者调用 `quality-coordinator`，后者调用 `code-reviewer`。

**为什么失败：**
- 每一层都增加延迟和 Token，但无决策价值
- 调试变成多层次的调查
- 叶子 Persona 因多次总结步骤而丢失上下文

**替代方案：** 保持编排深度最多为 1（slash command → Persona）。合并在主 Agent 中进行。

---

## Decision Flow（决策流程）

当考虑新的编排工作流时，按此流程走：

```
工作是否是对一个制品的一个视角？
├── 是 → 直接调用。停止。
└── 否 → 相同组合是否反复出现？
         ├── 否 → 直接调用，临时。停止。
         └── 是 → 子任务是否独立？
                  ├── 否 → 用户运行的顺序 slash command（模式 4）。
                  └── 是 → 并行扇出与合并（模式 3）。
                          对照上方清单进行验证。
                          如果任何检查失败 → 退回单一 Persona command（模式 2）。
```

---

## When to Add a New Pattern to This Catalog（何时向本目录添加新模式）

仅在满足以下条件时添加新条目：

1. 你已在实际工作中至少使用了该模式两次
2. 你能指出本仓库中的一个具体制品来展示它
3. 你能解释为什么现有模式本来不行
4. 你能描述其反模式的影子（人们会错误地构建什么替代它）

过早的目录条目会变成无人遵循的愿望文档。
