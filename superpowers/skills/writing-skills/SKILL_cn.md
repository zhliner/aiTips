---
name: writing-skills
description: 在创建新技能、编辑现有技能或部署前验证技能是否有效时使用
---

# 编写技能

## 概述

**编写技能就是将测试驱动开发（TDD）应用于流程文档。**

**个人技能存放在你的运行时的技能目录中**

你编写测试用例（使用 subagent 进行压力场景测试），观察它们失败（基线行为），编写技能（文档），观察测试通过（agent 遵守规则），然后重构（堵上漏洞）。

**核心原则：** 如果你没有亲眼看到一个 agent 在没有技能的情况下失败，你就不知道技能是否教了正确的东西。

**必需的背景知识：** 在使用本技能之前，你**必须**理解 superpowers:test-driven-development。该技能定义了基本的 RED-GREEN-REFACTOR 循环。本技能将 TDD 适配到文档工作中。

**官方指南：** 关于 Anthropic 官方的技能编写最佳实践，请参阅 anthropic-best-practices.md。该文档提供了额外的模式和指南，与本技能中注重 TDD 的方法相辅相成。

## 什么是技能？

**技能**是经过验证的技术、模式或工具的参考指南。技能帮助未来的 agent 发现并应用有效的方法。

**技能是：** 可复用的技术、模式、工具、参考指南

**技能不是：** 关于你曾经如何解决某个问题的叙述

## 技能的 TDD 映射

| TDD 概念 | 技能创建 |
|-------------|----------------|
| **测试用例** | 使用 subagent 进行压力场景测试 |
| **生产代码** | 技能文档（SKILL.md） |
| **测试失败（RED）** | Agent 在没有技能的情况下违反规则（基线） |
| **测试通过（GREEN）** | Agent 在有技能的情况下遵守规则 |
| **重构** | 在保持合规的同时堵上漏洞 |
| **先写测试** | 在编写技能**之前**运行基线场景 |
| **观察它失败** | 记录 agent 使用的确切借口 |
| **最小化代码** | 编写针对这些具体违规行为的技能 |
| **观察它通过** | 验证 agent 现在遵守规则 |
| **重构循环** | 发现新的借口 → 堵上 → 重新验证 |

整个技能创建过程遵循 RED-GREEN-REFACTOR。

## 何时创建技能

**在以下情况下创建：**
- 技术对你来说并非直观明显
- 你会在不同项目中再次引用它
- 模式适用范围广（不限于特定项目）
- 他人也会受益

**不要为以下情况创建：**
- 一次性解决方案
- 已在其他地方有完善文档的标准实践
- 项目特定的约定（放到你的 instructions 文件中）
- 机制性约束（如果可以用 regex/校验来强制执行，就自动化处理 —— 把文档留给判断性决策）

## 技能类型

### Technique（技术）
包含具体步骤的方法（condition-based-waiting、root-cause-tracing）

### Pattern（模式）
思考问题的方式（flatten-with-flags、test-invariants）

### Reference（参考）
API 文档、语法指南、工具文档（office docs）

## 目录结构


\`\`\`
skills/
  skill-name/
    SKILL.md              # 主要参考（必需）
    supporting-file.*     # 仅在需要时
\`\`\`

**扁平命名空间** - 所有技能在一个可搜索的命名空间中

**为以下内容使用独立文件：**
1. **大型参考**（100 行以上）- API 文档、综合语法
2. **可复用工具** - 脚本、实用工具、模板

**保持内联：**
- 原则和概念
- 代码模式（< 50 行）
- 其他所有内容

## SKILL.md 结构

**Frontmatter（YAML）：**
- 两个必填字段：`name` 和 `description`（所有支持的字段参见 [agentskills.io/specification](https://agentskills.io/specification)）
- 总共最多 1024 个字符
- `name`：仅使用字母、数字和连字符（不要使用括号和特殊字符）
- `description`：第三人称，仅描述**何时**使用（而非技能**做了什么**）
  - 以 "Use when..." 开头，聚焦触发条件
  - 包含具体的症状、场景和上下文
  - **绝不要**总结技能的过程或工作流（原因见 SDO 部分）
  - 尽可能保持在 500 字符以内

\`\`\`markdown
---
name: Skill-Name-With-Hyphens
description: Use when [具体的触发条件和症状]
---

# 技能名称

## Overview（概述）
这是什么？用 1-2 句说明核心原则。

## When to Use（使用时机）
[如果决策不显而易见，加入小型内联流程图]

使用症状和用例的项目符号列表
何时**不要**使用

## Core Pattern（核心模式）（用于技术/模式类技能）
修改前/修改后的代码对比

## Quick Reference（快速参考）
用于快速浏览常见操作的表格式或项目符号

## Implementation（实现）
简单模式的内联代码
对于大型参考或可复用工具，链接到独立文件

## Common Mistakes（常见错误）
什么会出错 + 如何修复

## Real-World Impact（实际效果）（可选）
具体成果
\`\`\`


## 技能发现优化（SDO）

**对发现至关重要：** 未来的 agent 需要**找到**你的技能

### 1. 丰富的描述字段

**目的：** 你的 agent 读取描述来决定为给定任务加载哪些技能。使其能回答："我现在应该读这个技能吗？"

**格式：** 以 "Use when..." 开头，聚焦触发条件

**关键：描述 = 何时使用，而非技能做了什么**

描述应该**只**描述触发条件。不要在描述中总结技能的过程或工作流。

**为什么这很重要：** 测试表明，当描述总结了技能的工作流时，agent 可能会按照描述去执行，而不是读取完整的技能内容。一个写着 "code review between tasks" 的描述导致 agent 只做**一次**审查，而技能中的流程图清楚地显示了**两次**审查（先规范合规性，再代码质量）。

当描述改为仅为 "Use when executing implementation plans with independent tasks"（没有工作流总结）时，agent 正确地读取了流程图并遵循了两阶段审查流程。

**陷阱：** 总结工作流的描述创建了一个 agent 会走的捷径。技能正文变成了 agent 跳过的文档。

\`\`\`yaml
# ❌ 坏：总结了工作流 —— agent 可能会按照这个去执行而不读取技能
description: Use when executing plans - dispatches subagent per task with code review between tasks

# ❌ 坏：太多的流程细节
description: Use for TDD - write test first, watch it fail, write minimal code, refactor

# ✅ 好：仅有触发条件，没有工作流总结
description: Use when executing implementation plans with independent tasks in the current session

# ✅ 好：仅有触发条件
description: Use when implementing any feature or bugfix, before writing implementation code
\`\`\`

**内容：**
- 使用具体的触发条件、症状和场景来表示该技能适用
- 描述**问题**（竞争条件、不一致行为）而非**语言特定的症状**（setTimeout、sleep）
- 保持触发条件与技术无关，除非技能本身是技术特定的
- 如果技能是技术特定的，在触发条件中明确说明
- 使用第三人称（注入到系统提示词中）
- **绝不要**总结技能的过程或工作流

\`\`\`yaml
# ❌ 坏：太抽象、模糊、未包含使用时机
description: For async testing

# ❌ 坏：第一人称
description: I can help you with async tests when they're flaky

# ❌ 坏：提到了技术但技能并非特定于它
description: Use when tests use setTimeout/sleep and are flaky

# ✅ 好：以 "Use when" 开头，描述问题，没有工作流
description: Use when tests have race conditions, timing dependencies, or pass/fail inconsistently

# ✅ 好：技术特定技能，具有明确的触发条件
description: Use when using React Router and handling authentication redirects
\`\`\`

### 2. 关键词覆盖

使用 agent 会搜索的词语：
- 错误消息："Hook timed out""ENOTEMPTY""race condition"
- 症状："flaky""hanging""zombie""pollution"
- 同义词："timeout/hang/freeze""cleanup/teardown/afterEach"
- 工具：实际的命令、库名称、文件类型

### 3. 描述性命名

**使用主动语态，动词优先：**
- ✅ `creating-skills` 而非 `skill-creation`
- ✅ `condition-based-waiting` 而非 `async-test-helpers`

### 4. Token 效率（关键）

**问题：** getting-started 和常用技能会加载到**每个**对话中。每个 token 都很重要。

**目标字数：**
- getting-started 工作流：每条 <150 词
- 频繁加载的技能：总计 <200 词
- 其他技能：<500 词（仍然要简洁）

**技巧：**

**将细节移到工具帮助中：**
\`\`\`bash
# ❌ 坏：在 SKILL.md 中记录所有标志
search-conversations supports --text, --both, --after DATE, --before DATE, --limit N

# ✅ 好：引用 --help
search-conversations supports multiple modes and filters. Run --help for details.
\`\`\`

**使用交叉引用：**
\`\`\`markdown
# ❌ 坏：重复工作流细节
When searching, dispatch subagent with template...
[20 行重复的指令]

# ✅ 好：引用其他技能
Always use subagents (50-100x context savings). REQUIRED: Use [other-skill-name] for workflow.
\`\`\`

**精简示例：**
\`\`\`markdown
# ❌ 坏：冗长示例（42 词）
your human partner: "How did we handle authentication errors in React Router before?"
You: I'll search past conversations for React Router authentication patterns.
[Dispatch subagent with search query: "React Router authentication error handling 401"]

# ✅ 好：精简示例（20 词）
Partner: "How did we handle auth errors in React Router?"
You: Searching...
[Dispatch subagent → synthesis]
\`\`\`

**消除冗余：**
- 不要重复交叉引用技能中的内容
- 不要解释命令中显而易见的内容
- 不要包含同一模式的多个示例

**验证：**
\`\`\`bash
wc -w skills/path/SKILL.md
# getting-started 工作流：目标每条 <150
# 其他常用：目标总计 <200
\`\`\`

**按其功能或核心洞察命名：**
- ✅ `condition-based-waiting` > `async-test-helpers`
- ✅ `using-skills` 而非 `skill-usage`
- ✅ `flatten-with-flags` > `data-structure-refactoring`
- ✅ `root-cause-tracing` > `debugging-techniques`

**动名词（-ing）形式很适合流程：**
- `creating-skills`、`testing-skills`、`debugging-with-logs`
- 主动态，描述你正在执行的操作

### 5. 交叉引用其他技能

**在编写引用其他技能的文档时：**

仅使用技能名称，并附带明确的必选标记：
- ✅ 好：`**REQUIRED SUB-SKILL:** Use superpowers:test-driven-development`
- ✅ 好：`**REQUIRED BACKGROUND:** You MUST understand superpowers:systematic-debugging`
- ❌ 坏：`See skills/testing/test-driven-development`（不明确是否必需）
- ❌ 坏：`@skills/testing/test-driven-development/SKILL.md`（强制加载，消耗上下文）

**为什么不使用 @ 链接：** `@` 语法会立即强制加载文件，在你需要之前就消耗 200k+ 上下文。

## 流程图的使用

\`\`\`dot
digraph when_flowchart {
    "Need to show information?" [shape=diamond];
    "Decision where I might go wrong?" [shape=diamond];
    "Use markdown" [shape=box];
    "Small inline flowchart" [shape=box];

    "Need to show information?" -> "Decision where I might go wrong?" [label="yes"];
    "Decision where I might go wrong?" -> "Small inline flowchart" [label="yes"];
    "Decision where I might go wrong?" -> "Use markdown" [label="no"];
}
\`\`\`

**仅对以下情况使用流程图：**
- 非显而易见的决策点
- 可能过早停止的流程循环
- "何时使用 A vs B"的决策

**绝不要对以下情况使用流程图：**
- 参考材料 → 表格、列表
- 代码示例 → Markdown 代码块
- 线性指令 → 编号列表
- 无语义含义的标签（step1、helper2）

关于 graphviz 样式规则，参见本目录中的 `graphviz-conventions.dot`。

**为你的合作伙伴可视化：** 使用本目录中的 `render-graphs.js` 将技能的流程图渲染为 SVG：
\`\`\`bash
./render-graphs.js ../some-skill           # 每个图分别渲染
./render-graphs.js ../some-skill --combine # 所有图合并在一个 SVG 中
\`\`\`

## 代码示例

**一个优秀的示例胜过许多平庸的示例**

选择最相关的语言：
- 测试技术 → TypeScript/JavaScript
- 系统调试 → Shell/Python
- 数据处理 → Python

**好的示例：**
- 完整且可运行
- 有良好的注释来解释**为什么**
- 来自真实场景
- 清楚地展示模式
- 可直接适配（非通用模板）

**不要：**
- 用 5 种以上语言实现
- 创建填空式模板
- 编写人为构造的示例

你很擅长移植 —— 一个优秀的示例就够了。

## 文件组织

### 自包含技能
\`\`\`
defense-in-depth/
  SKILL.md    # 所有内容内联
\`\`\`
何时使用：所有内容都适合内联，无需大型参考

### 带可复用工具的技能
\`\`\`
condition-based-waiting/
  SKILL.md    # 概述 + 模式
  example.ts  # 可适配的工作辅助代码
\`\`\`
何时使用：工具是可复用的代码，而非仅是叙述

### 带大型参考的技能
\`\`\`
pptx/
  SKILL.md       # 概述 + 工作流
  pptxgenjs.md   # 600 行 API 参考
  ooxml.md       # 500 行 XML 结构
  scripts/       # 可执行工具
\`\`\`
何时使用：参考材料太大，不适合内联

## 铁律（与 TDD 相同）

\`\`\`
NO SKILL WITHOUT A FAILING TEST FIRST
（没有先失败的测试，就不要创建技能）
\`\`\`

这适用于**新**技能和**现有**技能的**编辑**。

在测试前编写技能？删除它。重新开始。
编辑技能前没有测试？同样的违规。

**没有例外：**
- 不适用于"简单的添加"
- 不适用于"只是添加一节"
- 不适用于"文档更新"
- 不要把未测试的更改当作"参考"保留
- 不要在运行测试时"调整"
- 删除意味着删除

**必需的背景：** superpowers:test-driven-development 技能解释了为什么这很重要。同样的原则适用于文档。

## 各类技能的测试

不同类型的技能需要不同的测试方法：

### 纪律执行类技能（规则/要求）

**示例：** TDD、verification-before-completion、designing-before-coding

**测试方式：**
- 学术性问题：他们理解规则吗？
- 压力场景：他们在压力下遵守吗？
- 多重压力结合：时间 + 沉没成本 + 疲惫
- 识别借口并添加明确的驳回

**成功标准：** Agent 在最大压力下遵循规则

### 技术类技能（使用指南）

**示例：** condition-based-waiting、root-cause-tracing、defensive-programming

**测试方式：**
- 应用场景：他们能正确应用技术吗？
- 变体场景：他们处理边界情况吗？
- 信息缺失测试：指令有缺失吗？

**成功标准：** Agent 成功将技术应用到新场景

### 模式类技能（心智模型）

**示例：** reducing-complexity、information-hiding 概念

**测试方式：**
- 识别场景：他们识别出模式适用的时机吗？
- 应用场景：他们能使用心智模型吗？
- 反例：他们知道何时**不**使用吗？

**成功标准：** Agent 正确识别何时/如何应用模式

### 参考类技能（文档/API）

**示例：** API 文档、命令参考、库指南

**测试方式：**
- 检索场景：他们能找到正确的信息吗？
- 应用场景：他们能正确使用找到的信息吗？
- 覆盖测试：常见用例都覆盖了吗？

**成功标准：** Agent 找到并正确应用参考信息

## 跳过测试的常见借口

| 借口 | 现实 |
|--------|---------|
| "技能显然很清楚" | 对你清楚 ≠ 对其他 agent 清楚。测试它。 |
| "这只是一个参考" | 参考可能有遗漏、不清晰的部分。测试检索。 |
| "测试小题大做了" | 未测试的技能总有问题的。总是。15 分钟的测试节省数小时。 |
| "出问题时我会测试" | 出问题 = agent 无法使用技能。在部署**前**测试。 |
| "测试太繁琐了" | 测试比在生产环境中调试坏的技能更不繁琐。 |
| "我确信它很好" | 过度自信保证出问题。无论如何要测试。 |
| "学术审查就够了" | 阅读 ≠ 使用。测试应用场景。 |
| "没时间测试" | 部署未测试的技能浪费更多时间后续修复它。 |

**所有这些都意味着：部署前测试。没有例外。**

## 将形式与失败类型匹配

在编写指南之前，对基线失败进行分类。一种失败类型的防弹形式可能在另一种失败类型上产生可测量的反效果。

| 基线失败 | 正确形式 | 错误形式 |
|---|---|---|
| 在压力下跳过/违反规则（知道更好，还是这样做） | Prohibition（禁令）+ 借口表格 + 危险信号（见下方防弹部分） | 软性指南（"prefer...""consider..."） |
| 遵守规则但输出形状错误（臃肿的提示词、埋没的结论、复述规范） | 正面的配方或契约：说明输出**是什么** —— 其部分、按顺序 | 禁令列表（"don't restate""never narrate"） |
| 从他们已经在做的事情中遗漏必需元素 | 结构性的：在其填写的模板中使用 REQUIRED 字段或插槽 | 靠近模板的散文式提醒 |
| 行为应取决于条件 | 以可观察谓词为条件的条件性规则（"if the brief exists, reference it"） | 无条件规则 + 豁免条款 |

**为什么禁令在塑形问题上适得其反：** 在竞争性激励下（"使提示词自包含"），agent 会与"不要 X"进行谈判。在 dispatch-prompt 指南的正面交锋措辞测试中，禁令组产生的多余内容明显多于配方组（完全分离的分布），且比无指南对照组表现更差 —— 对你自己的情况做微测试而非假设，但永远不要默认使用禁令。配方不留谈判空间：输出要么匹配指定形状，要么不匹配。

**无论你选择哪种形式的规则：**
- **不要添加细微条款。** "Don't X unless it matters"重新开启了谈判 —— 在同样的措辞测试中，向一个成功的配方附加一条细微条款，使其从一致退化为有噪声。将真正的例外表达为基于可观察谓词的条件。
- **豁免条款无法限定范围。** "This limit doesn't apply to code blocks" 仍然会抑制代码块。如果输出的某部分必须豁免，重构使得规则无法触及它。

## 防止借口的技能防弹化

强制执行纪律的技能（如 TDD）需要抵抗找借口行为。Agent 很聪明，在压力下会找到漏洞。

**范围：** 此工具包适用于纪律性失败 —— agent 知道规则但在压力下跳过它。对于形状错误的输出或遗漏的元素，基于禁令的防弹化适得其反；请使用上文"将形式与失败类型匹配"中的形式。

**心理学说明：** 理解说服技术**为什么**有效，有助于你系统地应用它们。关于权威、承诺、稀缺性、社会认同和统一性原则的研究基础，请参阅 persuasion-principles.md（Cialdini, 2021; Meincke et al., 2025）。

### 明确堵住每一个漏洞

不要只声明规则 —— 明确禁止具体的变通方法：

<Bad>
\`\`\`markdown
在测试前就写了代码？删除它。
\`\`\`
</Bad>

<Good>
\`\`\`markdown
在测试前就写了代码？删除它。重新开始。

**没有例外：**
- 不要把它当作"参考"保留
- 不要在写测试时"调整"它
- 不要看它
- 删除意味着删除
\`\`\`
</Good>

### 回应"精神 vs 字面"的争论

在早期添加基本原则：

\`\`\`markdown
**违反规则的字面就是违反规则的精神。**
\`\`\`

这会切断整类"我遵循的是精神"的借口。

### 构建借口表格

从基线测试中捕获借口（见下方测试部分）。Agent 的每个借口都放入表格：

\`\`\`markdown
| 借口 | 现实 |
|--------|---------|
| "太简单不需要测试" | 简单的代码也会出错。测试只需 30 秒。 |
| "我之后再测试" | 之后通过的测试什么都证明不了。 |
| "测试在后也能达到同样目的" | 测试在后 = "这个做了什么？" 测试在先 = "这个应该做什么？" |
\`\`\`

### 创建危险信号列表

让 agent 在找借口时容易自我检查：

\`\`\`markdown
## 危险信号 —— 停下来，重新开始

- 测试前就写了代码
- "我已经手动测试过了"
- "之后测试能达到同样目的"
- "这是关于精神而非仪式"
- "这次不同是因为……"

**所有这些意味着：删除代码。用 TDD 重新开始。**
\`\`\`

### 为违规症状更新 SDO

添加到描述中：你**即将**违反规则时的症状：

\`\`\`yaml
description: use when implementing any feature or bugfix, before writing implementation code
\`\`\`

## 技能的 RED-GREEN-REFACTOR

遵循 TDD 循环：

### RED：编写失败测试（基线）

在**没有**技能的情况下运行带有 subagent 的压力场景。记录确切的行为：
- 他们做了什么选择？
- 他们使用了什么借口（逐字记录）？
- 哪些压力触发了违规？

这就是"观察测试失败"—— 在编写技能之前，你必须看到 agent 自然会做什么。

### GREEN：编写最精简的技能

编写针对这些特定借口的技能。不要为假设情况添加额外内容。

使用**有**技能的情况运行相同场景。Agent 现在应该遵守。

### REFACTOR：堵上漏洞

Agent 找到了新借口？添加明确的驳回。重新测试直到防弹。

### 在完整场景之前微测措辞

完整的压力场景运行是最终关卡，但每次迭代又慢又贵。先用微测试验证措辞本身：

1. **每次调用一个全新上下文样本** —— 原始 API 调用，或如果你没有 API 访问权限则使用一次性的 subagent。系统提示词 = 指南将存在的现实上下文（完整技能或 prompt 模板，而非孤立指南）；用户消息 = 一个诱导失败的任务。
2. **始终包含无指南对照组。** 如果对照组没有表现出失败，那就没有什么要修复的 —— 停止，不要编写指南。
3. **每个变体 5 次以上重复。** 单样本不可靠。
4. **手动读取每个标记的匹配。** 如果你愿意可以编程评分，但模板回显和引用的反例会伪装成命中；仅靠自动化计数会夸大失败和成功。
5. **方差是一个指标。** 当指南生效时，重复结果会收敛于相同的形状。五次重复中有五种不同的解释意味着措辞不具约束力 —— 在添加更多词之前收紧形式。

微测试验证措辞；它们不能替代纪律类技能的压力场景。

**测试方法：** 关于完整的测试方法，参见 [testing-skills-with-subagents.md](testing-skills-with-subagents.md)：
- 如何编写压力场景
- 压力类型（时间、沉没成本、权威、疲惫）
- 系统性堵漏洞
- 元测试技术

## 反模式

### ❌ 叙事性示例
"In session 2025-10-03, we found empty projectDir caused..."
**为什么不好：** 太具体、不可复用

### ❌ 多语言稀释
example-js.js、example-py.py、example-go.go
**为什么不好：** 质量平庸、维护负担

### ❌ 流程图中的代码
\`\`\`dot
step1 [label="import fs"];
step2 [label="read file"];
\`\`\`
**为什么不好：** 无法复制粘贴、难以阅读

### ❌ 通用标签
helper1、helper2、step3、pattern4
**为什么不好：** 标签应该有语义含义

## 停下来：在进入下一个技能之前

**编写完任何技能后，你必须停下来并完成部署流程。**

**不要：**
- 批量创建多个技能而不逐一测试
- 在当前技能验证完成前进入下一个
- 因为"批量更高效"而跳过测试

**下面的部署清单对每个技能都是强制性的。**

部署未测试的技能 = 部署未测试的代码。这违反了质量标准。

## 技能创建清单（TDD 适配版）

**重要：为下面的每一个清单项创建一个 todo。**

**RED 阶段 —— 编写失败测试：**
- [ ] 创建压力场景（纪律类技能需要 3 种以上组合压力）
- [ ] 在**没有**技能的情况下运行场景 —— 逐字记录基线行为
- [ ] 识别借口/失败中的模式

**GREEN 阶段 —— 编写最精简的技能：**
- [ ] 名称仅使用字母、数字、连字符（无括号/特殊字符）
- [ ] 包含必填 `name` 和 `description` 字段的 YAML frontmatter（最多 1024 字符；参见[规范](https://agentskills.io/specification)）
- [ ] 描述以 "Use when..." 开头，包含具体的触发条件和症状
- [ ] 描述使用第三人称
- [ ] 全文有关键词搜索覆盖（错误、症状、工具）
- [ ] 有清晰的概述和核心原则
- [ ] 回应了 RED 阶段识别的具体基线失败
- [ ] 指南形式与失败类型匹配（见"将形式与失败类型匹配"）
- [ ] 对于行为塑形指南：对照无指南对照组微测了措辞（5 次以上重复，每个标记匹配手动读取）—— 纯参考技能不适用
- [ ] 代码内联或链接到独立文件
- [ ] 一个优秀的示例（不是多语言）
- [ ] 使用有技能的情况运行场景 —— 验证 agent 现在遵守

**REFACTOR 阶段 —— 堵上漏洞：**
- [ ] 识别测试中出现的新借口
- [ ] 添加明确的驳回（如果是纪律类技能）
- [ ] 从所有测试迭代中构建借口表格
- [ ] 创建危险信号列表
- [ ] 重新测试直到防弹

**质量检查：**
- [ ] 仅在决策不显而易见时使用小型流程图
- [ ] 快速参考表格
- [ ] 常见错误部分
- [ ] 无叙事性讲述
- [ ] 辅助文件仅用于工具或大型参考

**部署：**
- [ ] 将技能提交到 git 并推送到你的 fork（如已配置）
- [ ] 如果广泛有用，考虑通过 PR 贡献回去

## 发现工作流

未来 agent 如何找到你的技能：

1. **遇到问题**（"测试不稳定"）
2. **搜索技能**（grep 描述、浏览分类）
3. **找到技能**（描述匹配）
4. **扫描概述**（这相关吗？）
5. **读取模式**（快速参考表格）
6. **加载示例**（仅在实现时）

**为此流程优化** —— 将可搜索的术语尽可能地放在前面和频繁出现的位置。

## 底线

**创建技能就是对流程文档应用 TDD。**

同样的铁律：没有先失败的测试，就没有技能。
同样的循环：RED（基线）→ GREEN（编写技能）→ REFACTOR（堵上漏洞）。
同样的好处：更好的质量、更少的意外、防弹的结果。

如果你对代码遵循 TDD，那么对技能也遵循它。这是应用于文档的同一纪律。
