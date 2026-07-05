---
name: skill-creator
description: 创建新 skill，修改和改进现有 skill，以及衡量 skill 性能。当用户想要从零开始创建 skill、编辑或优化现有 skill、运行 eval 测试 skill、通过方差分析对 skill 进行基准测试，或优化 skill 的 description 以提高触发准确性时使用。
---

# Skill Creator（技能创建器）

一个用于创建新 skill 并迭代改进它们的 skill。

从整体来看，创建 skill 的过程如下：

- 确定你希望 skill 做什么，以及大致的实现方式
- 编写 skill 的初稿
- 创建一些测试 prompt，并在带有该 skill 的 claude 上运行它们
- 帮助用户从定性和定量两个方面评估结果
  - 在后台运行测试的同时，如果还没有定量 eval，就起草一些（如果已有，可以直接使用，或者在认为需要调整时进行修改）。然后向用户解释这些 eval（如果已有现成的，则解释已有的内容）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果供其查看，同时让他们查看定量指标
- 根据用户评估结果的反馈（以及定量基准测试中暴露的明显缺陷）重写 skill
- 重复以上步骤，直到满意为止
- 扩展测试集，进行更大规模的再次测试

你使用此 skill 的工作是判断用户处于这个流程的哪个阶段，然后介入帮助他们推进。例如，用户可能会说"我想做一个关于 X 的 skill"。你可以帮助他们缩小范围、编写初稿、编写测试用例、确定评估方式、运行所有 prompt，然后迭代。

另一方面，用户可能已经有了 skill 的初稿。在这种情况下，你可以直接进入 eval/迭代环节。

当然，你应该始终保持灵活，如果用户说"我不需要运行一堆 eval，跟我随意聊聊就好"，你也可以那样做。

然后在 skill 完成之后（但同样，顺序是灵活的），你还可以运行 skill description 优化器——我们有一个专门的脚本来优化 skill 的触发效果。

明白了？好的。

## 与用户沟通

skill creator 的使用者可能对编程术语的熟悉程度差异很大。如果你还没听说（这也难怪，最近才开始），现在有一个趋势：Claude 的强大能力正在激励水管工打开他们的终端，父母和祖父母们去搜索"如何安装 npm"。另一方面，大部分用户可能对计算机相当熟悉。

所以请注意上下文线索来理解如何措辞你的沟通！在默认情况下，给你一些参考：

- "evaluation"和"benchmark"属于边界用语，但可以使用
- 对于"JSON"和"assertion"，你需要从用户那里看到明确的线索表明他们知道这些概念，才能在不解释的情况下使用

如果不确定，可以简短地解释术语，如果你不确定用户是否能理解，可以用简短的定义加以说明。

---

## 创建 skill

### 捕获意图

首先理解用户的意图。当前对话中可能已经包含了用户想要捕获的工作流程（例如，他们说"把这个变成一个 skill"）。如果是这样，先从对话历史中提取答案——使用的工具、步骤顺序、用户做出的修正、观察到的输入/输出格式。用户可能需要补充空白，并在进入下一步之前确认。

1. 这个 skill 应该让 Claude 能够做什么？
2. 这个 skill 应该在什么时候触发？（哪些用户短语/上下文）
3. 期望的输出格式是什么？
4. 我们是否应该设置测试用例来验证 skill 是否有效？具有客观可验证输出的 skill（文件转换、数据提取、代码生成、固定工作流步骤）适合使用测试用例。具有主观输出的 skill（写作风格、艺术）通常不需要。根据 skill 类型建议适当的默认值，但让用户决定。

### 访谈与调研

主动询问关于边界情况、输入/输出格式、示例文件、成功标准和依赖项的问题。在这部分确定之前，先不要编写测试 prompt。

检查可用的 MCP——如果对调研有用（搜索文档、查找类似的 skill、查阅最佳实践），可以通过 subagent 并行调研（如果可用），否则内联进行。准备好上下文以减少用户的负担。

### 编写 SKILL.md

根据用户访谈，填写以下组件：

- **name**：skill 标识符
- **description**：何时触发，做什么。这是主要的触发机制——既要包含 skill 的功能，也要包含使用它的具体上下文。所有"何时使用"的信息都放在这里，而不是正文中。注意：目前 Claude 倾向于"触发不足"——在 skill 有用时不使用它们。为了解决这个问题，请让 skill 的 description 稍微"主动"一些。例如，不要写"如何构建一个简单快速的仪表板来显示 Anthropic 内部数据。"，你可以写"如何构建一个简单快速的仪表板来显示 Anthropic 内部数据。确保在用户提到仪表板、数据可视化、内部指标，或想要展示任何类型的公司数据时使用此 skill，即使他们没有明确要求'dashboard'。"
- **compatibility**：所需工具、依赖项（可选，很少需要）
- **skill 的其余部分 :)**

### Skill 编写指南

#### Skill 的结构

```
skill-name/
├── SKILL.md（必需）
│   ├── YAML frontmatter（name、description 必需）
│   └── Markdown 指令
└── 捆绑资源（可选）
    ├── scripts/    - 用于确定性/重复性任务的可执行代码
    ├── references/ - 按需加载到上下文中的文档
    └── assets/     - 输出中使用的文件（模板、图标、字体）
```

#### 渐进式披露

Skill 使用三级加载系统：
1. **元数据**（name + description）- 始终在上下文中（约 100 个词）
2. **SKILL.md 正文** - skill 触发时即在上下文中（理想情况下不超过 500 行）
3. **捆绑资源** - 按需加载（无限制，脚本可以在不加载的情况下执行）

这些字数是近似值，如果需要可以更长。

**关键模式：**
- 将 SKILL.md 控制在 500 行以内；如果接近此限制，添加额外的层次结构，并清楚地指出使用该 skill 的模型接下来应该去哪里跟进。
- 从 SKILL.md 中清楚地引用参考文件，并说明何时应该阅读它们
- 对于大型参考文件（>300 行），包含目录

**领域组织**：当一个 skill 支持多个领域/框架时，按变体组织：
```
cloud-deploy/
├── SKILL.md（工作流 + 选择）
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```
Claude 只读取相关的参考文件。

#### 无意外原则

这不言而喻，但 skill 不得包含恶意软件、漏洞利用代码或任何可能危害系统安全的内容。如果被描述，skill 的内容不应让用户对其意图感到意外。不要配合创建误导性 skill 或旨在促进未授权访问、数据泄露或其他恶意活动的 skill 的请求。不过，像"扮演一个 XYZ"之类的请求是可以的。

#### 编写模式

在指令中优先使用祈使句。

**定义输出格式** - 你可以这样做：
```markdown
## 报告结构
始终使用此精确模板：
# [标题]
## 执行摘要
## 关键发现
## 建议
```

**示例模式** - 包含示例很有用。你可以这样格式化（但如果示例中有"Input"和"Output"，你可能需要稍作调整）：
```markdown
## 提交消息格式
**示例 1：**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 写作风格

尝试向模型解释为什么某些事情很重要，而不是使用强硬的"必须"。运用心智理论，尝试让 skill 通用化，而不是过于局限于特定示例。先写一个初稿，然后用全新的视角审视并改进它。

### 测试用例

编写 skill 初稿后，构思 2-3 个真实的测试 prompt——真实用户实际会说的那种话。与用户分享：[你不必使用这个精确的措辞]"这里有几个我想尝试的测试用例。这些看起来合适吗，还是你想添加更多？"然后运行它们。

将测试用例保存到 `evals/evals.json`。先不要写 assertion——只写 prompt。你将在下一步中利用运行进行中的时间起草 assertion。

```json
{
  "skill_name": "example-skill",
  "evals": [
    {
      "id": 1,
      "prompt": "用户的任务 prompt",
      "expected_output": "期望结果的描述",
      "files": []
    }
  ]
}
```

完整的 schema（包括你稍后会添加的 `assertions` 字段），请参见 `references/schemas.md`。

## 运行和评估测试用例

本节是一个连续的序列——不要中途停下来。不要使用 `/skill-test` 或任何其他测试 skill。

将结果放在 `<skill-name>-workspace/` 中，作为 skill 目录的兄弟目录。在工作区内，按迭代组织结果（`iteration-1/`、`iteration-2/` 等），在每个迭代内，每个测试用例有一个目录（`eval-0/`、`eval-1/` 等）。不要预先创建所有这些——在过程中按需创建目录。

### 步骤 1：在同一轮中生成所有运行（with-skill 和 baseline）

对于每个测试用例，在同一轮中生成两个 subagent——一个带 skill，一个不带。这很重要：不要先生成 with-skill 运行，然后再回来做 baseline。一次性启动所有任务，这样它们大约同时完成。

**With-skill 运行：**

```
执行此任务：
- Skill 路径：<path-to-skill>
- 任务：<eval prompt>
- 输入文件：<eval 文件（如有），或 "none">
- 将输出保存到：<workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- 要保存的输出：<用户关心的内容——例如，".docx 文件"、"最终的 CSV">
```

**Baseline 运行**（相同的 prompt，但 baseline 取决于上下文）：
- **创建新 skill**：完全不使用 skill。相同的 prompt，没有 skill 路径，保存到 `without_skill/outputs/`。
- **改进现有 skill**：使用旧版本。在编辑之前，快照 skill（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后将 baseline subagent 指向快照。保存到 `old_skill/outputs/`。

为每个测试用例编写一个 `eval_metadata.json`（assertion 暂时可以为空）。给每个 eval 一个描述性的名称，基于它测试的内容——而不仅仅是"eval-0"。目录也使用这个名称。如果此迭代使用了新的或修改过的 eval prompt，为每个新的 eval 目录创建这些文件——不要假设它们会从之前的迭代继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "用户的任务 prompt",
  "assertions": []
}
```

### 步骤 2：在运行进行中起草 assertion

不要只是等待运行完成——你可以利用这段时间做有成效的事情。为每个测试用例起草定量 assertion，并向用户解释。如果 `evals/evals.json` 中已有 assertion，审查它们并解释它们检查什么。

好的 assertion 是客观可验证的，并且有描述性的名称——它们应该在 benchmark 查看器中清晰可读，让浏览结果的人能立即理解每个 assertion 检查什么。主观 skill（写作风格、设计质量）更适合定性评估——不要将 assertion 强加于需要人类判断的事物上。

起草完成后，用 assertion 更新 `eval_metadata.json` 文件和 `evals/evals.json`。同时向用户解释他们将在查看器中看到什么——包括定性输出和定量 benchmark。

### 步骤 3：运行完成时，捕获时间数据

当每个 subagent 任务完成时，你会收到一个包含 `total_tokens` 和 `duration_ms` 的通知。立即将此数据保存到运行目录中的 `timing.json`：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获此数据的唯一机会——它通过任务通知传来，不会在其他地方持久化。在收到每个通知时立即处理，而不是尝试批量处理。

### 步骤 4：评分、汇总并启动查看器

所有运行完成后：

1. **为每次运行评分** — 生成一个 grader subagent（或内联评分），读取 `agents/grader.md` 并根据输出评估每个 assertion。将结果保存到每个运行目录中的 `grading.json`。grading.json 的 expectations 数组必须使用 `text`、`passed` 和 `evidence` 字段（不是 `name`/`met`/`details` 或其他变体）——查看器依赖这些确切的字段名。对于可以通过编程方式检查的 assertion，编写并运行脚本而不是目视检查——脚本更快、更可靠，并且可以跨迭代重用。

2. **汇总为 benchmark** — 从 skill-creator 目录运行汇总脚本：
   ```bash
   python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
   ```
   这会生成 `benchmark.json` 和 `benchmark.md`，包含每种配置的 pass_rate、time 和 tokens，以及 mean ± stddev 和 delta。如果手动生成 benchmark.json，请参见 `references/schemas.md` 了解查看器期望的确切 schema。
将每个 with_skill 版本放在其 baseline 对应版本之前。

3. **进行分析** — 读取 benchmark 数据并揭示汇总统计可能隐藏的模式。请参见 `agents/analyzer.md`（"分析 Benchmark 结果"部分）了解需要关注什么——例如无论是否使用 skill 都始终通过的 assertion（无区分度）、高方差 eval（可能不稳定），以及时间/token 权衡。

4. **启动查看器**，同时展示定性输出和定量数据：
   ```bash
   nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
   ```
   对于第 2 次及以后的迭代，还需传入 `--previous-workspace <workspace>/iteration-<N-1>`。

   **Cowork / 无头环境：** 如果 `webbrowser.open()` 不可用或环境没有显示器，使用 `--static <output_path>` 写入一个独立的 HTML 文件，而不是启动服务器。当用户点击"Submit All Reviews"时，反馈将下载为 `feedback.json` 文件。下载后，将 `feedback.json` 复制到工作区目录中，供下一次迭代使用。

注意：请使用 generate_review.py 创建查看器；无需编写自定义 HTML。

5. **告诉用户**类似这样的话："我已经在你的浏览器中打开了结果。有两个标签页——'Outputs'让你点击查看每个测试用例并留下反馈，'Benchmark'显示定量比较。完成后，回到这里告诉我。"

### 用户在查看器中看到什么

"Outputs"标签页一次显示一个测试用例：
- **Prompt**：给出的任务
- **Output**：skill 生成的文件，尽可能内联渲染
- **Previous Output**（第 2 次及以后的迭代）：折叠区域，显示上次迭代的输出
- **Formal Grades**（如果运行了评分）：折叠区域，显示 assertion 通过/失败
- **Feedback**：一个文本框，输入时自动保存
- **Previous Feedback**（第 2 次及以后的迭代）：他们上次的评论，显示在文本框下方

"Benchmark"标签页显示统计摘要：每种配置的通过率、时间和 token 使用量，以及每个 eval 的详细分解和分析师观察。

导航通过 prev/next 按钮或方向键。完成后，点击"Submit All Reviews"，将所有反馈保存到 `feedback.json`。

### 步骤 5：读取反馈

当用户告诉你他们完成了，读取 `feedback.json`：

```json
{
  "reviews": [
    {"run_id": "eval-0-with_skill", "feedback": "图表缺少坐标轴标签", "timestamp": "..."},
    {"run_id": "eval-1-with_skill", "feedback": "", "timestamp": "..."},
    {"run_id": "eval-2-with_skill", "feedback": "完美，我很喜欢这个", "timestamp": "..."}
  ],
  "status": "complete"
}
```

空反馈意味着用户认为没问题。将改进重点放在用户有具体意见的测试用例上。

完成后关闭查看器服务器：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 改进 skill

这是迭代循环的核心。你已经运行了测试用例，用户已经审查了结果，现在你需要根据他们的反馈让 skill 变得更好。

### 如何思考改进

1. **从反馈中归纳。** 这里的大局是，我们试图创建可以被使用一百万次（也许真的是，也许更多谁知道呢）跨越许多不同 prompt 的 skill。在这里，你和用户只在少数几个示例上反复迭代，因为这样推进更快。用户对这些示例了如指掌，可以快速评估新的输出。但如果你和用户共同开发的 skill 只对这些示例有效，那它就是没用的。与其做琐碎的过拟合修改，或压制性的严格"必须"，如果有某个顽固的问题，你可以尝试拓展思路，使用不同的比喻，或推荐不同的工作模式。尝试的成本相对较低，也许你会找到很棒的方案。

2. **保持 prompt 精简。** 移除没有发挥作用的内容。确保阅读执行记录（transcript），而不仅仅是最终输出——如果看起来 skill 让模型浪费了大量时间做无效的事情，你可以尝试去掉 skill 中导致这种情况的部分，看看会发生什么。

3. **解释原因。** 尽力解释你要求模型做的每件事背后的**原因**。今天的大语言模型非常*聪明*。它们有良好的心智理论，当给予好的框架时，能够超越死板的指令，真正发挥作用。即使用户的反馈简短或沮丧，也要尝试真正理解任务、理解用户为什么写了他们写的内容、他们实际写了什么，然后将这种理解传递到指令中。如果你发现自己在写全大写的 ALWAYS 或 NEVER，或使用超级僵化的结构，那是一个黄色警告——如果可能的话，重新构建并解释推理，让模型理解你要求的东西为什么重要。这是一种更人性化、更强大、更有效的方法。

4. **寻找跨测试用例的重复工作。** 阅读测试运行的执行记录，注意 subagent 是否都独立编写了类似的辅助脚本或对某些事情采取了相同的多步骤方法。如果所有 3 个测试用例都导致 subagent 编写了 `create_docx.py` 或 `build_chart.py`，这是一个强烈的信号，表明 skill 应该捆绑该脚本。写一次，放在 `scripts/` 中，并告诉 skill 使用它。这可以避免每次未来的调用都重新发明轮子。

这项任务非常重要（我们在这里努力创造每年数十亿美元的经济价值！），你的思考时间不是瓶颈；花时间好好思考。我建议写一个修改草案，然后用全新的眼光审视并做出改进。真正尽你所能进入用户的思维，理解他们想要和需要什么。

### 迭代循环

改进 skill 后：

1. 将改进应用到 skill
2. 将所有测试用例重新运行到新的 `iteration-<N+1>/` 目录中，包括 baseline 运行。如果你正在创建新 skill，baseline 始终是 `without_skill`（无 skill）——这在各迭代中保持不变。如果你正在改进现有 skill，根据你的判断决定什么作为 baseline 最合理：用户最初带来的原始版本，还是上一次迭代。
3. 使用 `--previous-workspace` 指向上一次迭代来启动查看器
4. 等待用户审查并告诉你他们完成了
5. 读取新的反馈，再次改进，重复

持续进行直到：
- 用户说他们满意了
- 反馈全部为空（一切看起来都好）
- 你没有取得有意义的进展

---

## 进阶：盲评比较

在需要更严格地比较两个版本 skill 的情况下（例如，用户问"新版本真的更好吗？"），有一个盲评比较系统。阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思路是：将两个输出交给一个独立的 agent，不告诉它哪个是哪个，让它判断质量。然后分析获胜者为什么获胜。

这是可选的，需要 subagent，大多数用户不需要它。人工审查循环通常就足够了。

---

## Description 优化

SKILL.md frontmatter 中的 description 字段是决定 Claude 是否调用 skill 的主要机制。创建或改进 skill 后，提供优化 description 以提高触发准确性的服务。

### 步骤 1：生成触发 eval 查询

创建 20 个 eval 查询——混合应该触发和不应该触发的。保存为 JSON：

```json
[
  {"query": "用户的 prompt", "should_trigger": true},
  {"query": "另一个 prompt", "should_trigger": false}
]
```

查询必须是真实的，是 Claude Code 或 Claude.ai 用户实际会输入的内容。不是抽象的请求，而是具体、详细、有足够细节的请求。例如，文件路径、关于用户工作或情况的个人上下文、列名和值、公司名称、URL。一些背景故事。有些可能是小写的，或包含缩写、拼写错误或口语化表达。使用不同长度的混合，并专注于边界情况而不是让它们非黑即白（用户将有机会审核它们）。

不好的例子：`"Format this data"`、`"Extract text from PDF"`、`"Create a chart"`

好的例子：`"ok so my boss just sent me this xlsx file (its in my downloads, called something like 'Q4 sales final FINAL v2.xlsx') and she wants me to add a column that shows the profit margin as a percentage. The revenue is in column C and costs are in column D i think"`

对于**应该触发**的查询（8-10 个），考虑覆盖率。你需要同一意图的不同措辞——有些正式，有些随意。包含用户没有明确提到 skill 或文件类型但显然需要它的情况。加入一些不常见的使用场景，以及这个 skill 与另一个 skill 竞争但应该获胜的情况。

对于**不应该触发**的查询（8-10 个），最有价值的是那些接近但未命中的——与 skill 共享关键词或概念但实际上需要不同处理的查询。想想相邻领域、天真的关键词匹配会触发但不应该触发的模糊措辞，以及查询涉及 skill 所做的事情但在另一个工具更合适的上下文中的情况。

要避免的关键问题：不要让不应该触发的查询明显无关。用"Write a fibonacci function"作为 PDF skill 的负面测试太简单了——它什么都测不出来。负面案例应该真正具有挑战性。

### 步骤 2：与用户审查

使用 HTML 模板向用户展示 eval 集供审查：

1. 从 `assets/eval_review.html` 读取模板
2. 替换占位符：
   - `__EVAL_DATA_PLACEHOLDER__` → eval 项的 JSON 数组（不要加引号——它是一个 JS 变量赋值）
   - `__SKILL_NAME_PLACEHOLDER__` → skill 的名称
   - `__SKILL_DESCRIPTION_PLACEHOLDER__` → skill 当前的 description
3. 写入临时文件（例如 `/tmp/eval_review_<skill-name>.html`）并打开：`open /tmp/eval_review_<skill-name>.html`
4. 用户可以编辑查询、切换 should-trigger、添加/删除条目，然后点击"Export Eval Set"
5. 文件下载到 `~/Downloads/eval_set.json`——如果有多个版本（例如 `eval_set (1).json`），检查 Downloads 文件夹中最新的版本

这一步很重要——糟糕的 eval 查询会导致糟糕的 description。

### 步骤 3：运行优化循环

告诉用户："这需要一些时间——我将在后台运行优化循环，并定期检查进度。"

将 eval 集保存到工作区，然后在后台运行：

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用系统提示中的 model ID（驱动当前会话的那个），这样触发测试与用户实际体验相匹配。

在运行期间，定期查看输出，告诉用户当前在第几次迭代以及分数情况。

这会自动处理完整的优化循环。它将 eval 集分为 60% 的训练集和 40% 的保留测试集，评估当前 description（每个查询运行 3 次以获得可靠的触发率），然后调用 Claude 根据失败情况提出改进建议。它在训练集和测试集上重新评估每个新 description，最多迭代 5 次。完成后，它在浏览器中打开一个 HTML 报告，显示每次迭代的结果，并返回包含 `best_description` 的 JSON——根据测试集分数而非训练集分数选择，以避免过拟合。

### Skill 触发的工作原理

理解触发机制有助于设计更好的 eval 查询。Skill 出现在 Claude 的 `available_skills` 列表中，带有 name 和 description，Claude 根据 description 决定是否查阅 skill。需要知道的重要一点是，Claude 只会为它自己无法轻松处理的任务查阅 skill——简单的单步查询如"read this PDF"可能不会触发 skill，即使 description 完美匹配，因为 Claude 可以用基础工具直接处理它们。复杂的、多步骤的或专业化的查询在 description 匹配时会可靠地触发 skill。

这意味着你的 eval 查询应该足够实质性，使 Claude 真正能从查阅 skill 中受益。简单的查询如"read file X"是糟糕的测试用例——无论 description 质量如何，它们都不会触发 skill。

### 步骤 4：应用结果

从 JSON 输出中取 `best_description` 并更新 skill 的 SKILL.md frontmatter。向用户展示前后对比并报告分数。

---

### 打包和展示（仅在 `present_files` 工具可用时）

检查你是否可以访问 `present_files` 工具。如果没有，跳过此步骤。如果有，打包 skill 并将 .skill 文件展示给用户：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包后，引导用户到生成的 `.skill` 文件路径，以便他们可以安装它。

---

## Claude.ai 特定说明

在 Claude.ai 中，核心工作流程相同（起草 → 测试 → 审查 → 改进 → 重复），但由于 Claude.ai 没有 subagent，一些机制需要改变。以下是需要调整的内容：

**运行测试用例**：没有 subagent 意味着不能并行执行。对于每个测试用例，读取 skill 的 SKILL.md，然后按照其指令自己完成测试 prompt。逐个进行。这不如独立的 subagent 严格（你写了 skill 又在运行它，所以你有完整的上下文），但这是一个有用的健全性检查——而且人工审查步骤可以弥补。跳过 baseline 运行——直接使用 skill 完成请求的任务。

**审查结果**：如果你无法打开浏览器（例如，Claude.ai 的虚拟机没有显示器，或者你在远程服务器上），完全跳过浏览器查看器。相反，直接在对话中展示结果。对于每个测试用例，显示 prompt 和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），保存到文件系统并告诉他们位置，以便他们可以下载和检查。内联征求反馈："这看起来怎么样？有什么想改的吗？"

**基准测试**：跳过定量基准测试——它依赖于在没有 subagent 的情况下没有意义的 baseline 比较。专注于用户的定性反馈。

**迭代循环**：与之前相同——改进 skill、重新运行测试用例、征求反馈——只是中间没有浏览器查看器。如果你有文件系统，仍然可以将结果组织到迭代目录中。

**Description 优化**：此部分需要 `claude` CLI 工具（特别是 `claude -p`），仅在 Claude Code 中可用。如果你在 Claude.ai 上，跳过它。

**盲评比较**：需要 subagent。跳过。

**打包**：`package_skill.py` 脚本在任何有 Python 和文件系统的地方都能工作。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

**更新现有 skill**：用户可能要求你更新现有 skill，而不是创建新的。在这种情况下：
- **保留原始名称。** 记下 skill 的目录名和 `name` frontmatter 字段——保持不变。例如，如果安装的 skill 是 `research-helper`，输出 `research-helper.skill`（不是 `research-helper-v2`）。
- **在编辑前复制到可写位置。** 已安装的 skill 路径可能是只读的。复制到 `/tmp/skill-name/`，在那里编辑，然后从副本打包。
- **如果手动打包，先暂存在 `/tmp/`**，然后复制到输出目录——直接写入可能因权限而失败。

---

## Cowork 特定说明

如果你在 Cowork 中，需要知道的主要事项是：

- 你有 subagent，所以主工作流程（并行生成测试用例、运行 baseline、评分等）都能工作。（但是，如果你遇到严重的超时问题，可以串行运行测试 prompt 而不是并行。）
- 你没有浏览器或显示器，所以在生成 eval 查看器时，使用 `--static <output_path>` 写入一个独立的 HTML 文件，而不是启动服务器。然后提供一个链接，让用户可以在浏览器中打开 HTML。
- 出于某种原因，Cowork 的设置似乎让 Claude 在运行测试后不愿意生成 eval 查看器，所以再次强调：无论你在 Cowork 还是 Claude Code 中，运行测试后，你都应该始终生成 eval 查看器，让人类在你自己审查并尝试修正之前查看示例，使用 `generate_review.py`（不要编写你自己的定制 HTML 代码）。提前抱歉，我要用大写了：在*自己*评估输入之前，先生成 EVAL 查看器。你要尽快把它们展示给人类！
- 反馈的工作方式不同：因为没有运行的服务器，查看器的"Submit All Reviews"按钮会将 `feedback.json` 下载为文件。然后你可以从那里读取它（你可能需要先请求访问权限）。
- 打包可以工作——`package_skill.py` 只需要 Python 和文件系统。
- Description 优化（`run_loop.py` / `run_eval.py`）应该在 Cowork 中正常工作，因为它通过 subprocess 使用 `claude -p`，而不是浏览器，但请等到你完全完成 skill 制作并且用户同意它状态良好后再进行。
- **更新现有 skill**：用户可能要求你更新现有 skill，而不是创建新的。请遵循上面 Claude.ai 部分中的更新指南。

---

## 参考文件

agents/ 目录包含专门 subagent 的指令。在需要生成相应 subagent 时阅读它们。

- `agents/grader.md` — 如何根据输出评估 assertion
- `agents/comparator.md` — 如何在两个输出之间进行盲评 A/B 比较
- `agents/analyzer.md` — 如何分析一个版本为什么击败了另一个

references/ 目录有额外的文档：
- `references/schemas.md` — evals.json、grading.json 等的 JSON 结构

---

再次重复核心循环以强调：

- 弄清楚 skill 是关于什么的
- 起草或编辑 skill
- 在测试 prompt 上运行带有该 skill 的 claude
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 帮助用户审查
  - 运行定量 eval
- 重复直到你和用户都满意
- 打包最终的 skill 并返回给用户。

如果你有 TodoList，请添加步骤以确保不会忘记。如果你在 Cowork 中，请特别将"创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人类审查测试用例"放入你的 TodoList 以确保执行。

祝你好运！
