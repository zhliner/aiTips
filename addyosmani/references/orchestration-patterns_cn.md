# 编排模式（Orchestration Patterns）

本仓库推荐的 Agent 编排模式参考目录，以及需要避免的反模式。在添加一个协调多个 persona 的新 slash command 之前，或者在引入一个"包装"现有 persona 的新 persona 之前，请先阅读本文。

核心规则：**用户（或 slash command）是编排者。Persona 不能调用其他 persona。** Skill 是 persona 工作流中的必经环节。

---

## 推荐模式

### 1. 直接调用（无编排）

单个 persona，单个视角，单个产出物。这是默认且成本最低的方式。

```
user → code-reviewer → report → user
```

**适用场景：** 工作是对单个产出物的单一视角分析，且能用一句话描述。

**示例：**
- "Review this PR" → `code-reviewer`
- "Find security issues in `auth.ts`" → `security-auditor`
- "What tests are missing for the checkout flow?" → `test-engineer`

**成本：** 一次往返。这是你始终应该用来对比编排模式的基线。

---

### 2. 单 persona slash command

一个 slash command 将单个 persona 与项目的 skill 组合在一起。免去用户每次重复解释工作流的麻烦。

```
/review → code-reviewer（配合 code-review-and-quality skill）→ report
```

**适用场景：** 相同的单 persona 调用以相同的配置反复出现。

**本仓库中的示例：** `/review`、`/test`、`/code-simplify`。

**成本：** 与直接调用相同。Slash command 只是一个保存的 prompt。

**警示信号：** 如果 slash command 的主体大部分是"决定调用哪个 persona"，那就删掉它，让用户直接调用 persona。

---

### 3. 并行扇出与合并（Parallel fan-out with merge）

多个 persona 并发处理同一输入，各自生成独立的报告。一个合并步骤（在主 agent 的上下文中）将它们综合为单一决策。

```
                    ┌─→ code-reviewer    ─┐
/ship → fan out  ───┼─→ security-auditor ─┤→ merge → go/no-go + rollback
                    └─→ test-engineer    ─┘
```

**适用场景：**
- 子任务之间真正独立（无共享可变状态，无顺序依赖）
- 每个子 agent 能从自己的上下文窗口中获益
- 合并步骤足够小，可以在主上下文中完成
- 实际等待时间很重要

**本仓库中的示例：** `/ship`。

**成本：** N 个并行子 agent 上下文 + 一次合并轮次。高于直接调用，但实际耗时更短，且因为每个子 agent 专注于单一视角，报告质量更高。

**采用此模式前的验证清单：**
- [ ] 我能否同时运行所有子 agent 而不存在顺序问题？
- [ ] 每个 persona 是否产出不同*类型*的发现，而不仅仅是从不同角度得出相同发现？
- [ ] 合并步骤能否放入主 agent 的剩余上下文中？
- [ ] 用户的等待时间是否足够长，使得并行化效果明显可感知？

如果任何一项答案为"否"，请回退到直接调用或单 persona command。

---

### 4. 用户驱动的序列化 slash command 流水线

用户按定义好的顺序运行 slash command，在它们之间传递上下文（或 commit 历史）。没有编排 agent——用户**就是**编排者。

```
user runs:  /spec  →  /plan  →  /build  →  /test  →  /review  →  /ship
```

**适用场景：** 工作流存在依赖（每一步需要上一步的输出），且步骤之间的人工判断能带来价值。

**本仓库中的示例：** 整个 DEFINE → PLAN → BUILD → VERIFY → REVIEW → SHIP 生命周期。

**成本：** 每步一个子 agent 上下文。编排层本身无成本，因为没有编排 agent。

**为什么不自动化：** 一个 LLM "生命周期编排器"会（a）因需要为交接而做摘要导致步骤间细节丢失，（b）跳过能在早期发现方向性错误的人工检查点，（c）因释义轮次导致 token 成本翻倍。

---

### 5. 研究隔离（上下文保护）

当任务需要阅读大量材料，而这些材料不应污染主上下文时，生成一个研究子 agent，只返回摘要。

```
main agent → research sub-agent（读取 50 个文件）→ digest → main agent 继续
```

**适用场景：**
- 主会话需要保持对下游任务的聚焦
- 调研结果远小于其消耗的输入
- 主 agent 在之后有足够空间思考，从而提高决策质量

**示例：** "找出 monorepo 中这个已废弃 API 的所有调用点"、"总结这 30 个 ADR 关于缓存的结论。"

**成本：** 一个隔离的子 agent 上下文。只要替代方案是将数百个文件加载到主上下文中，就值得使用。

**在 Claude Code 上，使用内置的 `Explore` subagent**，而不是定义自定义研究 persona。`Explore` 运行在 Haiku 上，被禁止使用写入/编辑工具，且专为此模式而设计。只有在 `Explore` 不适用时（例如，你需要一个领域特定的 system prompt，而模型无法自行推断）才定义自定义研究子 agent。

---

## Claude Code 兼容性

本目录与具体工具无关，但大多数读者会在 Claude Code 上运行它。以下是每种模式如何映射到 Claude Code 的原语——以及平台在哪里替我们强制执行规则。

### Persona 的存放位置

插件子 agent 放在插件根目录的 `agents/` 中。本仓库是一个插件（`.claude-plugin/plugin.json`），因此 `agents/code-reviewer.md`、`agents/security-auditor.md` 和 `agents/test-engineer.md` 在插件启用时会被自动发现，无需路径配置。

### Subagent 与 Agent Teams

Claude Code 有两种并行化原语。模式 3（并行扇出与合并）映射到 **subagent**。如果你需要能互相交流的队友，则使用 **Agent Teams**。

| | Subagent | Agent Teams |
|--|-----------|-------------|
| 协调方式 | 主 agent 扇出，子 agent 仅回报结果 | 队友之间互相发消息，共享任务列表 |
| 上下文 | 每个子 agent 拥有独立的上下文窗口 | 每个队友拥有独立的上下文窗口 |
| 使用时机 | 独立任务，产出报告 | 需要讨论的协作性工作 |
| 状态 | 稳定 | 实验性——需要 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` |
| 成本 | 较低 | 较高——每个队友是一个独立的 Claude 实例 |

**本仓库中的 persona 在两种模式下都能工作。** 当作为子 agent 生成时（例如由 `/ship`），它们将发现报告给主会话。当作为队友生成时（`Spawn a teammate using the security-auditor agent type…`），它们可以直接质疑彼此的发现。Persona 定义是相同的；只有生成上下文不同。

一个细节：persona 中的 `skills` 和 `mcpServers` frontmatter 字段在作为子 agent 运行时会被遵守，但在**作为队友运行时会被忽略**——队友从你的项目和用户设置中加载 skill 和 MCP server，与普通会话相同。如果一个 persona 依赖特定的 skill 或 MCP server 被加载，请在会话级别进行配置，以便在两种模式下都可用。

### 平台强制规则

本目录中的两条规则不仅仅是约定——Claude Code 会强制执行它们：

- **"Subagents cannot spawn other subagents"**（文档原文）。反模式 B（persona 调用 persona）和反模式 D（深层 persona 树）在 Claude Code 上从结构上就不可能存在。
- **"No nested teams"**——队友不能生成自己的团队。同样的反模式在团队层面被阻止。

这意味着你可以放心采用本目录中的模式，不必担心贡献者意外构建出反模式。它们只会加载失败。

### 需要了解的内置子 agent

在定义自定义子 agent 之前，先检查以下内置的是否能满足需求：

| 内置子 agent | 用途 |
|----------|---------|
| `Explore` | 只读的代码库搜索与分析。用于模式 5（研究隔离）。 |
| `Plan` | 计划模式下的只读研究。 |
| `general-purpose` | 需要探索和修改的多步骤任务。 |

不要重新定义这些。将你的专家 persona（code-reviewer、security-auditor、test-engineer）构建在它们之上。

### 插件 agent 的 Frontmatter 限制

插件子 agent **不**支持 `hooks`、`mcpServers` 或 `permissionMode` frontmatter 字段——这些会被静默忽略。如果未来的 persona 需要其中任何一项，用户必须将文件复制到 `.claude/agents/` 或 `~/.claude/agents/` 中。

插件 agent 中**可用**的字段有：`name`、`description`、`tools`、`disallowedTools`、`model`、`maxTurns`、`skills`、`memory`、`background`、`effort`、`isolation`、`color`、`initialPrompt`。如果你希望优化成本，可以按 persona 使用 `model`（例如，Haiku 用于 `test-engineer` 的覆盖率扫描，Sonnet 用于 `code-reviewer`，Opus 用于 `security-auditor`）。

### 并行生成多个子 agent

在 Claude Code 中，并行扇出（模式 3）需要在**单个 assistant 轮次中发出多个 Agent 工具调用**。序列化轮次会导致执行串行化。`/ship` 明确指出了这一点。任何新的编排命令也应如此。

---

## 实战示例：使用 Agent Teams 进行竞争假设调试

本示例展示何时应该使用 **Agent Teams** 而非 `/ship` 的子 agent 扇出。这两种模式从远处看很相似——都生成相同的三个 persona——但价值来源不同。

### 场景

> *Checkout 偶尔会挂起约 30 秒后才完成。大约每 50 次会话出现一次。日志中没有错误。始于上周的发布。*

合理的根因（互斥，均符合症状）：

1. 新的支付确认流程中存在竞态条件
2. 某个 auth 检查偶尔穿透到缓慢的同步网络调用
3. 某个查询缺少索引，其性能随购物车大小缩放
4. 某个不稳定的第三方 API，SDK 在超时前会静默重试

单个 agent 会选择第一个合理的假设并停止调查。`/ship` 风格的子 agent 扇出让每个 persona 独立报告——但它们的报告互不交叉，因此无法排除错误的假设。

这正是 Agent Teams 文档描述的场景：*"当多个独立调查者积极尝试推翻彼此的假设时，最终存活下来的假设更可能是真正的根因。"*

### 为什么这**不是** `/ship` 的工作

| | `/ship`（子 agent） | Agent Teams |
|--|--------------------|-------------|
| 子 agent 看到 | 相同的 diff，不同的视角 | 共享的任务列表，彼此的消息 |
| 输出 | 三份独立报告 → 一次合并 | 对抗性辩论 → 共识根因 |
| 适用时机 | 你需要对已知产出物做出裁决 | 你需要在假设中*找到*产出物 |

`/ship` 是裁决；Agent Teams 是调查。

### 设置（一次性，每个环境）

Agent Teams 是实验性的。在 `~/.claude/settings.json` 中：

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

需要 Claude Code v2.1.32 或更高版本。本仓库中的 persona 会被自动识别——无需手动编写团队配置文件。

### 触发 prompt

在主会话中用自然语言输入：

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

主会话生成三个队友，引用现有的 persona 名称。Persona body 会被**追加**到每个队友的 system prompt 中作为额外指令（在 lead 安装的团队协调指令之上）；上述触发 prompt 成为它们的任务。

### 执行过程

1. 每个队友在自己的上下文窗口中运行，从自己的视角探索代码库。
2. 队友使用 `message` 直接互相发送发现。Lead 无需中转。
3. 共享任务列表显示谁在调查什么——随时可通过 `Ctrl+T`（进程内模式）或 tmux 窗格（分屏模式）查看。
4. 当 `code-reviewer` 发现一个应该是顺序执行的 `Promise.all` 时，它会发消息给 `security-auditor` 确认 auth 调用是否属于竞态的一部分。`security-auditor` 检查后回复——要么确认竞态是真正的问题，要么提出反证。
5. `test-engineer` 为当前最有可能的假设提出一个有针对性的集成测试，团队用它来在宣布共识前进行验证。
6. Lead 综合收敛后的发现并向你展示。

你可以通过 `Shift+Down` 循环切换到任意队友并输入——这在重定向走了错误路径的调查者时很有用。

### 何时清理

当调查锁定根因后，告诉 lead：

```
Clean up the team
```

始终通过 lead 进行清理，而不是通过队友（根据文档：队友缺少完整的团队上下文来执行清理）。

### 成本预期

三个 Sonnet 队友运行约 10–15 分钟的调查，成本明显高于由 `/ship` 生成的相同三个 persona 子 agent。其合理性在于*结论质量*——对于错误修复代价高昂的生产环境调试，额外的 token 开销是值得的。对于常规 PR 审查，坚持使用 `/ship`。

### 此场景中的反模式

**不要**将其重建为一个扇出子 agent 的 `/debug` slash command。子 agent 之间无法通信——你会失去使该模式生效的对抗性辩论。如果一个工作流反复出现，请将上述触发 prompt 记录为代码片段，而不是将其包装成滥用子 agent 的 slash command。

### 何时**不**使用 Agent Teams

- 对已知 diff 的生产环境裁决 → 使用 `/ship`（子 agent）。
- 对单个产出物的单一专家视角 → 直接调用 persona。
- 序列化生命周期（spec → plan → build）→ 用户驱动的 slash command（模式 4）。
- 阅读量大但摘要小的研究 → 内置 `Explore` 子 agent。

只有在队友**需要**互相质疑才能得出正确答案时，才使用 Agent Teams。

---

## 反模式

### A. 路由 persona（"元编排器"）

一个负责决定调用哪个其他 persona 的 persona。

```
/work → router-persona → "this needs a review" → code-reviewer → router（释义）→ user
```

**为什么失败：**
- 纯粹的路由层，无领域价值
- 增加两次释义跳转 → 信息丢失 + 约 2 倍 token 成本
- 用户已经知道自己需要 review；他们本可以直接调用 `/review`
- 复制了 slash command 和 `AGENTS.md` 中意图映射已经完成的工作

**替代方案：** 添加或完善 slash command。在 `AGENTS.md` 中记录意图 → 命令的映射。

---

### B. Persona 调用另一个 Persona

一个 `code-reviewer` 在看到 auth 代码时内部调用 `security-auditor`。

**为什么失败：**
- Persona 的设计是产出单一视角；链式调用违背了这一设计
- 调用方 persona 传递的摘要会丢失被调用方 persona 所需的上下文
- 失败模式成倍增加（哪个 persona 的输出格式优先？谁的规则生效？）
- 对用户隐藏了成本

**替代方案：** 让调用方 persona 在其报告中*建议*后续审计。由用户或 slash command 执行第二轮检查。

---

### C. 释义式序列化编排器

一个代表用户依次调用 `/spec`、然后 `/plan`、然后 `/build` 等的 agent。

**为什么失败：**
- 丢失了能在早期发现方向性错误的人工检查点
- 每次交接都会摘要上下文——在长流水线中累积偏移
- token 成本翻倍：每一步都需要编排轮次 + 子 agent 轮次
- 在判断力最重要的环节剥夺了用户的主动权

**替代方案：** 保持用户作为编排者。在 `README.md` 中记录推荐序列，让用户自行调用。

---

### D. 深层 Persona 树

`/ship` 调用 `pre-ship-coordinator`，后者调用 `quality-coordinator`，后者再调用 `code-reviewer`。

**为什么失败：**
- 每一层都增加延迟和 token 消耗，却不产生决策价值
- 调试变成多层调查
- 叶子 persona 因多次摘要步骤丢失上下文

**替代方案：** 将编排深度保持在最多 1 层（slash command → persona）。合并在主 agent 中完成。

---

## 决策流程

在考虑新的编排工作流时，按以下流程判断：

```
工作是否是对单个产出物的单一视角？
├── 是 → 直接调用。结束。
└── 否  → 相同的组合是否会重复出现？
         ├── 否  → 直接调用，临时组合。结束。
         └── 是  → 子任务之间是否独立？
                  ├── 否  → 用户运行的序列化 slash command（模式 4）。
                  └── 是  → 并行扇出与合并（模式 3）。
                           按上述清单验证。
                           如果任何检查未通过 → 回退到单 persona command（模式 2）。
```

---

## 何时向本目录添加新模式

仅在满足以下条件后才添加新条目：

1. 你已在实际工作中至少使用过该模式两次
2. 你能指出本仓库中展示该模式的具体产出物
3. 你能解释为什么现有模式无法满足需求
4. 你能描述其反模式影子（人们会错误地构建出什么）

过早添加的目录条目会变成无人遵循的愿景文档。
