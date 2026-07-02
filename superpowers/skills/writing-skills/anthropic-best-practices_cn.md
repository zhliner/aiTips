# 技能编写最佳实践

> 学习如何编写有效的技能，让 agent 能够发现并成功使用。

好的技能简洁、结构良好，并经过真实使用场景的测试。本指南提供实用的编写决策，帮助你编写 agent 能够发现并有效使用的技能。

关于技能工作机制的概念背景，请参阅[技能概述](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)。

## 核心原则

### 简洁是关键

[上下文窗口](https://platform.claude.com/docs/en/build-with-claude/context-windows)是公共资源。你的技能与 agent 需要知道的所有其他内容共享上下文窗口，包括：

* 系统提示词
* 对话历史
* 其他技能的元数据
* 你实际的请求

并非技能中的每个 token 都有即时成本。启动时，只有所有技能的元数据（name 和 description）被预加载。Agent 仅在技能变得相关时读取 SKILL.md，并且仅在需要时读取其他文件。然而，在 SKILL.md 中保持简洁仍然很重要：一旦 agent 加载了它，每个 token 都与对话历史和其他上下文竞争。

**默认假设**：Agent 已经非常聪明

只添加 agent 尚不具备的上下文。对每条信息进行质疑：

* "agent 真的需要这个解释吗？"
* "我可以假设 agent 知道这个吗？"
* "这段值得它的 token 成本吗？"

**好的示例：简洁**（约 50 tokens）：

````markdown  theme={null}
## Extract PDF text

Use pdfplumber for text extraction:

```python
import pdfplumber

with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```
````

**坏的示例：过于冗长**（约 150 tokens）：

```markdown  theme={null}
## Extract PDF text

PDF (Portable Document Format) files are a common file format that contains
text, images, and other content. To extract text from a PDF, you will need to
use a library. There are many libraries available for PDF processing, but we
recommend pdfplumber because it is easy to use and handles most cases well.
First, you will need to install it using pip. Then you can use the code below...
```

简洁版本假设 agent 知道 PDF 是什么以及库是如何工作的。

### 设置适当的自由度

将具体性级别与任务的脆弱性和可变性相匹配。

**高自由度**（基于文本的指令）：

适用于：

* 多种方法都是有效的
* 决策取决于上下文
* 启发式方法指导方式

示例：

```markdown  theme={null}
## Code review process

1. Analyze the code structure and organization
2. Check for potential bugs or edge cases
3. Suggest improvements for readability and maintainability
4. Verify adherence to project conventions
```

**中自由度**（伪代码或带参数的脚本）：

适用于：

* 存在首选模式
* 允许一些变化
* 配置影响行为

示例：

````markdown  theme={null}
## Generate report

Use this template and customize as needed:

```python
def generate_report(data, format="markdown", include_charts=True):
    # Process data
    # Generate output in specified format
    # Optionally include visualizations
```
````

**低自由度**（特定脚本，参数很少或没有）：

适用于：

* 操作脆弱且容易出错
* 一致性至关重要
* 必须遵循特定顺序

示例：

````markdown  theme={null}
## Database migration

Run exactly this script:

```bash
python scripts/migrate.py --verify --backup
```

Do not modify the command or add additional flags.
````

**类比**：将 agent 想象为一个探索路径的机器人：

* **两边都是悬崖的窄桥**：只有一条安全的前进道路。提供具体的护栏和精确的指令（低自由度）。示例：必须按精确顺序运行的数据库迁移。
* **没有危险的开放田野**：多条路径通向成功。给出大致方向，信任 agent 找到最佳路径（高自由度）。示例：上下文决定最佳方法的代码审查。

### 使用你计划使用的所有模型进行测试

技能作为模型的补充，因此有效性取决于底层模型。使用你计划使用的所有模型来测试你的技能。

**按模型的测试考量**：

* **Claude Haiku**（快速、经济）：技能是否提供足够的指导？
* **Claude Sonnet**（平衡）：技能是否清晰高效？
* **Claude Opus**（强推理能力）：技能是否避免了过度解释？

对 Opus 效果完美的可能需要为 Haiku 提供更多细节。如果你计划在多个模型上使用你的技能，目标是指令在所有模型上都能良好工作。

## Skill 结构

<Note>
  **YAML Frontmatter**：SKILL.md 的 frontmatter 需要两个字段：

  * `name` - 技能的可读名称（最多 64 个字符）
  * `description` - 技能做什么以及何时使用的单行描述（最多 1024 个字符）

  完整的技能结构细节，请参见[技能概述](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#skill-structure)。
</Note>

### 命名规范

使用一致的命名模式，使技能更容易引用和讨论。我们建议对技能名称使用**动名词形式**（动词 + -ing），因为这清楚地描述了技能提供的活动或能力。

**好的命名示例（动名词形式）**：

* "Processing PDFs"
* "Analyzing spreadsheets"
* "Managing databases"
* "Testing code"
* "Writing documentation"

**可接受的替代方案**：

* 名词短语："PDF Processing""Spreadsheet Analysis"
* 行动导向："Process PDFs""Analyze Spreadsheets"

**避免**：

* 模糊的名称："Helper""Utils""Tools"
* 过于泛化："Documents""Data""Files"
* 技能集合中不一致的模式

一致的命名使得：
* 在文档和对话中引用技能更容易
* 一眼就能理解技能的功能
* 组织和搜索多个技能更容易
* 维护专业、连贯的技能库

### 编写有效的描述

`description` 字段支持技能发现，应包含技能做什么以及何时使用。

<Warning>
  **始终使用第三人称**。描述被注入到系统提示词中，不一致的视角可能导致发现问题。

  * **好：** "Processes Excel files and generates reports"
  * **避免：** "I can help you process Excel files"
  * **避免：** "You can use this to process Excel files"
</Warning>

**具体并包含关键术语**。同时包含技能做什么以及何时使用的特定触发条件/上下文。

每个技能恰好有一个描述字段。描述对技能选择至关重要：agent 用它从潜在的 100+ 可用技能中选择合适的技能。你的描述必须提供足够的细节让 agent 知道何时选择该技能，而 SKILL.md 的其余部分提供实现细节。

有效示例：

**PDF 处理技能：**

```yaml  theme={null}
description: Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
```

**Excel 分析技能：**

```yaml  theme={null}
description: Analyze Excel spreadsheets, create pivot tables, generate charts. Use when analyzing Excel files, spreadsheets, tabular data, or .xlsx files.
```

**Git 提交助手技能：**

```yaml  theme={null}
description: Generate descriptive commit messages by analyzing git diffs. Use when the user asks for help writing commit messages or reviewing staged changes.
```

避免像这样的模糊描述：

```yaml  theme={null}
description: Helps with documents
```

```yaml  theme={null}
description: Processes data
```

```yaml  theme={null}
description: Does stuff with files
```

### 渐进式披露模式

SKILL.md 作为概述，根据需要将 agent 引导到详细材料，就像入职指南中的目录一样。关于渐进式披露如何工作的解释，请参见概述中的[技能如何工作](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#how-skills-work)。

**实用指南：**

* 保持 SKILL.md 正文在 500 行以内以获得最佳性能
* 当接近该限制时，将内容拆分到独立文件中
* 使用以下模式有效地组织指令、代码和资源

#### 可视化概览：从简单到复杂

一个基本的技能从只包含元数据和指令的单个 SKILL.md 文件开始：

完整的技能目录结构可能如下所示：

```
pdf/
├── SKILL.md              # 主要指令（触发时加载）
├── FORMS.md              # 表单填写指南（按需加载）
├── reference.md          # API 参考（按需加载）
├── examples.md           # 使用示例（按需加载）
└── scripts/
    ├── analyze_form.py   # 实用脚本（执行，不加载）
    ├── fill_form.py      # 表单填写脚本
    └── validate.py       # 验证脚本
```

#### 模式 1：带引用的高级指南

````markdown  theme={null}
---
name: PDF Processing
description: Extracts text and tables from PDF files, fills forms, and merges documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
---

# PDF Processing

## Quick start

Extract text with pdfplumber:
```python
import pdfplumber
with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```

## Advanced features

**Form filling**: See [FORMS.md](FORMS.md) for complete guide
**API reference**: See [REFERENCE.md](REFERENCE.md) for all methods
**Examples**: See [EXAMPLES.md](EXAMPLES.md) for common patterns
````

Agent 仅在需要时加载 FORMS.md、REFERENCE.md 或 EXAMPLES.md。

#### 模式 2：领域特定组织

对于具有多个领域的技能，按领域组织内容以避免加载不相关的上下文。当用户询问销售指标时，agent 只需要读取与销售相关的 schema，而不是财务或营销数据。这保持 token 使用量低且上下文专注。

```
bigquery-skill/
├── SKILL.md (概述和导航)
└── reference/
    ├── finance.md (收入、账单指标)
    ├── sales.md (机会、pipeline)
    ├── product.md (API 使用、功能)
    └── marketing.md (活动、归因)
```

````markdown SKILL.md theme={null}
# BigQuery Data Analysis

## Available datasets

**Finance**: Revenue, ARR, billing -> See [reference/finance.md](reference/finance.md)
**Sales**: Opportunities, pipeline, accounts -> See [reference/sales.md](reference/sales.md)
**Product**: API usage, features, adoption -> See [reference/product.md](reference/product.md)
**Marketing**: Campaigns, attribution, email -> See [reference/marketing.md](reference/marketing.md)

## Quick search

Find specific metrics using grep:

```bash
grep -i "revenue" reference/finance.md
grep -i "pipeline" reference/sales.md
grep -i "api usage" reference/product.md
```
````

#### 模式 3：条件性细节

显示基本内容，链接到高级内容：

```markdown  theme={null}
# DOCX Processing

## Creating documents

Use docx-js for new documents. See [DOCX-JS.md](DOCX-JS.md).

## Editing documents

For simple edits, modify the XML directly.

**For tracked changes**: See [REDLINING.md](REDLINING.md)
**For OOXML details**: See [OOXML.md](OOXML.md)
```

Agent 仅在用户需要这些功能时读取 REDLINING.md 或 OOXML.md。

### 避免深层嵌套引用

当从其他引用文件引用时，agent 可能会部分读取文件。遇到嵌套引用时，agent 可能使用 `head -100` 等命令预览内容，而不是读取完整文件，导致信息不完整。

**将引用保持在 SKILL.md 的一层深度**。所有参考文件应直接从 SKILL.md 链接，以确保 agent 在需要时读取完整文件。

**坏的示例：太深**：

```markdown  theme={null}
# SKILL.md
See [advanced.md](advanced.md)...

# advanced.md
See [details.md](details.md)...

# details.md
Here is the actual information...
```

**好的示例：一层深度**：

```markdown  theme={null}
# SKILL.md

**Basic usage**: [instructions in SKILL.md]
**Advanced features**: See [advanced.md](advanced.md)
**API reference**: See [reference.md](reference.md)
**Examples**: See [examples.md](examples.md)
```

### 为较长的参考文件设计目录结构

对于超过 100 行的参考文件，在顶部包含目录。这确保 agent 即使在部分读取预览时也能看到可用信息的完整范围。

**示例**：

```markdown  theme={null}
# API Reference

## Contents
- Authentication and setup
- Core methods (create, read, update, delete)
- Advanced features (batch operations, webhooks)
- Error handling patterns
- Code examples

## Authentication and setup
...

## Core methods
...
```

Agent 然后可以根据需要读取完整文件或跳转到特定部分。关于这种基于文件系统的架构如何实现渐进式披露的详细信息，请参见下文高级部分中的运行时环境部分。

## 工作流和反馈循环

### 为复杂任务使用工作流

将复杂操作分解为清晰、顺序的步骤。对于特别复杂的工作流，提供一个清单，agent 可以将其复制到其响应中并随进度勾选。

**示例 1：研究综合工作流**（适用于没有代码的技能）：

````markdown  theme={null}
## Research synthesis workflow

Copy this checklist and track your progress:

```
Research Progress:
- [ ] Step 1: Read all source documents
- [ ] Step 2: Identify key themes
- [ ] Step 3: Cross-reference claims
- [ ] Step 4: Create structured summary
- [ ] Step 5: Verify citations
```

**Step 1: Read all source documents**

Review each document in the `sources/` directory. Note the main arguments and supporting evidence.

**Step 2: Identify key themes**

Look for patterns across sources. What themes appear repeatedly? Where do sources agree or disagree?

**Step 3: Cross-reference claims**

For each major claim, verify it appears in the source material. Note which source supports each point.

**Step 4: Create structured summary**

Organize findings by theme. Include:
- Main claim
- Supporting evidence from sources
- Conflicting viewpoints (if any)

**Step 5: Verify citations**

Check that every claim references the correct source document. If citations are incomplete, return to Step 3.
````

此示例展示了工作流如何适用于不需要代码的分析任务。清单模式适用于任何复杂的多步骤过程。

**示例 2：PDF 表单填写工作流**（适用于带代码的技能）：

````markdown  theme={null}
## PDF form filling workflow

Copy this checklist and check off items as you complete them:

```
Task Progress:
- [ ] Step 1: Analyze the form (run analyze_form.py)
- [ ] Step 2: Create field mapping (edit fields.json)
- [ ] Step 3: Validate mapping (run validate_fields.py)
- [ ] Step 4: Fill the form (run fill_form.py)
- [ ] Step 5: Verify output (run verify_output.py)
```

**Step 1: Analyze the form**

Run: `python scripts/analyze_form.py input.pdf`

This extracts form fields and their locations, saving to `fields.json`.

**Step 2: Create field mapping**

Edit `fields.json` to add values for each field.

**Step 3: Validate mapping**

Run: `python scripts/validate_fields.py fields.json`

Fix any validation errors before continuing.

**Step 4: Fill the form**

Run: `python scripts/fill_form.py input.pdf fields.json output.pdf`

**Step 5: Verify output**

Run: `python scripts/verify_output.py output.pdf`

If verification fails, return to Step 2.
````

清晰的步骤防止 agent 跳过关键的验证。清单帮助你和 agent 跟踪多步骤工作流的进度。

### 实现反馈循环

**常见模式**：运行验证器 -> 修复错误 -> 重复

此模式大大提高了输出质量。

**示例 1：风格指南合规**（适用于没有代码的技能）：

```markdown  theme={null}
## Content review process

1. Draft your content following the guidelines in STYLE_GUIDE.md
2. Review against the checklist:
   - Check terminology consistency
   - Verify examples follow the standard format
   - Confirm all required sections are present
3. If issues found:
   - Note each issue with specific section reference
   - Revise the content
   - Review the checklist again
4. Only proceed when all requirements are met
5. Finalize and save the document
```

这展示了使用参考文档而非脚本的验证循环模式。"验证器"是 STYLE_GUIDE.md，agent 通过阅读和比较来执行检查。

**示例 2：文档编辑过程**（适用于带代码的技能）：

```markdown  theme={null}
## Document editing process

1. Make your edits to `word/document.xml`
2. **Validate immediately**: `python ooxml/scripts/validate.py unpacked_dir/`
3. If validation fails:
   - Review the error message carefully
   - Fix the issues in the XML
   - Run validation again
4. **Only proceed when validation passes**
5. Rebuild: `python ooxml/scripts/pack.py unpacked_dir/ output.docx`
6. Test the output document
```

验证循环及早捕获错误。

## 内容指南

### 避免时间敏感的信息

不要包含会过时的信息：

**坏的示例：时间敏感的**（会变错）：

```markdown  theme={null}
If you are doing this before August 2025, use the old API.
After August 2025, use the new API.
```

**好的示例**（使用"old patterns"部分）：

```markdown  theme={null}
## Current method

Use the v2 API endpoint: `api.example.com/v2/messages`

## Old patterns

<details>
<summary>Legacy v1 API (deprecated 2025-08)</summary>

The v1 API used: `api.example.com/v1/messages`

This endpoint is no longer supported.
</details>
```

旧模式部分提供了历史背景而不使主要内容混乱。

### 使用一致的术语

选择一个术语并在整个技能中坚持使用：

**好的 -- 一致**：

* 始终用 "API endpoint"
* 始终用 "field"
* 始终用 "extract"

**坏的 -- 不一致**：

* 混用 "API endpoint""URL""API route""path"
* 混用 "field""box""element""control"
* 混用 "extract""pull""get""retrieve"

一致性帮助 agent 理解并遵循指令。

## 常见模式

### 模板模式

为输出格式提供模板。将严格程度与你的需求相匹配。

**对于严格的要求**（如 API 响应或数据格式）：

````markdown  theme={null}
## Report structure

ALWAYS use this exact template structure:

```markdown
# [Analysis Title]

## Executive summary
[One-paragraph overview of key findings]

## Key findings
- Finding 1 with supporting data
- Finding 2 with supporting data
- Finding 3 with supporting data

## Recommendations
1. Specific actionable recommendation
2. Specific actionable recommendation
```
````

**对于灵活的指导**（当适应有用时）：

````markdown  theme={null}
## Report structure

Here is a sensible default format, but use your best judgment based on the analysis:

```markdown
# [Analysis Title]

## Executive summary
[Overview]

## Key findings
[Adapt sections based on what you discover]

## Recommendations
[Tailor to the specific context]
```

Adjust sections as needed for the specific analysis type.
````

### 示例模式

对于输出质量依赖于看到示例的技能，提供输入/输出对，就像在常规提示中一样：

````markdown  theme={null}
## Commit message format

Generate commit messages following these examples:

**Example 1:**
Input: Added user authentication with JWT tokens
Output:
```
feat(auth): implement JWT-based authentication

Add login endpoint and token validation middleware
```

**Example 2:**
Input: Fixed bug where dates displayed incorrectly in reports
Output:
```
fix(reports): correct date formatting in timezone conversion

Use UTC timestamps consistently across report generation
```

**Example 3:**
Input: Updated dependencies and refactored error handling
Output:
```
chore: update dependencies and refactor error handling

- Upgrade lodash to 4.17.21
- Standardize error response format across endpoints
```

Follow this style: type(scope): brief description, then detailed explanation.
````

示例比仅靠描述更清楚地帮助 agent 理解所需的风格和详细程度。

### 条件工作流模式

引导 agent 通过决策点：

```markdown  theme={null}
## Document modification workflow

1. Determine the modification type:

   **Creating new content?** -> Follow "Creation workflow" below
   **Editing existing content?** -> Follow "Editing workflow" below

2. Creation workflow:
   - Use docx-js library
   - Build document from scratch
   - Export to .docx format

3. Editing workflow:
   - Unpack existing document
   - Modify XML directly
   - Validate after each change
   - Repack when complete
```

<Tip>
  如果工作流变得庞大或复杂且包含许多步骤，考虑将它们推送到独立文件中，并告诉 agent 根据手头任务读取适当的文件。
</Tip>

## 评估和迭代

### 首先构建评估

**在编写大量文档之前创建评估。** 这确保你的技能解决真实问题而非记录想象的问题。

**评估驱动开发：**

1. **识别差距**：在没有技能的情况下，在有代表性的任务上运行你的 agent。记录具体的失败或缺失的上下文
2. **创建评估**：构建三个测试这些差距的场景
3. **建立基线**：测量 agent 在没有技能时的表现
4. **编写最精简的指令**：创建刚好足够的内容来解决差距并通过评估
5. **迭代**：执行评估，与基线比较，并细化

这种方法确保你解决的是实际问题，而不是预测可能永远不会出现的需求。

**评估结构**：

```json  theme={null}
{
  "skills": ["pdf-processing"],
  "query": "Extract all text from this PDF file and save it to output.txt",
  "files": ["test-files/document.pdf"],
  "expected_behavior": [
    "Successfully reads the PDF file using an appropriate PDF processing library or command-line tool",
    "Extracts text content from all pages in the document without missing any pages",
    "Saves the extracted text to a file named output.txt in a clear, readable format"
  ]
}
```

<Note>
  此示例展示了一个带有简单测试评分标准的数据驱动评估。我们目前不提供运行这些评估的内置方式。用户可以创建自己的评估系统。评估是你衡量技能有效性的真实来源。
</Note>

### 与 agent 迭代开发技能

最有效的技能开发过程涉及 agent 本身。与一个实例（"Agent A"）合作创建一个将被其他实例（"Agent B"）使用的技能。Agent A 帮助你设计和细化指令，而 Agent B 在实际任务中测试它们。这是可行的，因为底层模型理解如何编写有效的 agent 指令以及 agent 需要什么信息。

**创建新技能：**

1. **在没有技能的情况下完成任务**：使用正常提示与 Agent A 解决一个问题。在此过程中，你自然会提供上下文、解释偏好并分享流程知识。注意你反复提供的信息。

2. **识别可复用的模式**：完成任务后，识别你提供的哪些上下文对类似的将来任务有用。

   **示例**：如果你进行了一次 BigQuery 分析，你可能提供了表名、字段定义、过滤规则（如"始终排除测试账户"）和常见查询模式。

3. **要求 Agent A 创建技能**："创建一个技能来捕捉我们刚使用的这个 BigQuery 分析模式。包括表 schema、命名约定以及关于过滤测试账户的规则。"

   <Tip>
     现代 agent 原生理解技能格式和结构。你不需要特殊的系统提示或"编写技能"的技能来获得创建技能的帮助。只需要求 agent 创建一个技能，它就会生成结构正确的 SKILL.md 内容，包含适当的 frontmatter 和正文内容。
   </Tip>

4. **审查简洁性**：检查 Agent A 是否添加了不必要的解释。要求："删除关于 win rate 含义的解释 -- agent 已经知道那个。"

5. **改进信息架构**：要求 Agent A 更有效地组织内容。例如："将其组织成表 schema 放在一个单独的参考文件中。我们以后可能添加更多表。"

6. **在类似任务上测试**：在相关的用例上使用 Agent B（一个加载了该技能的新实例）使用该技能。观察 Agent B 是否找到正确的信息、正确应用规则、成功处理任务。

7. **基于观察进行迭代**：如果 Agent B 遇到困难或遗漏了什么，带着具体信息返回 Agent A："当 agent 使用这个技能时，它忘了按 Q4 的日期过滤。我们应该添加一个关于日期过滤模式的部分吗？"

**迭代现有技能：**

同样的层级模式在改进技能时继续。你在以下两者之间交替：

* **与 Agent A 合作**（帮助细化技能的专家）
* **使用 Agent B 测试**（使用该技能执行实际工作的 agent）
* **观察 Agent B 的行为**并将洞察带回 Agent A

1. **在真实工作流中使用技能**：给 Agent B（加载了技能）实际的任务，而非测试场景

2. **观察 Agent B 的行为**：注意它在何处遇到困难、成功或做出意外的选择

   **示例观察**："当我要求 Agent B 做一个区域销售报告时，它写了查询但忘了过滤掉测试账户，即使技能提到了这个规则。"

3. **返回 Agent A 进行改进**：分享当前的 SKILL.md 并描述你观察到的。询问："我注意到当我要求做一个区域报告时，Agent B 忘了过滤测试账户。技能提到了过滤，但也许它不够突出？"

4. **审查 Agent A 的建议**：Agent A 可能会建议重组以使规则更突出，使用更强的语言如 "MUST filter" 而非 "always filter"，或重构工作流部分。

5. **应用并测试更改**：使用 Agent A 的细化更新技能，然后在类似的请求上再次使用 Agent B 测试

6. **基于使用重复**：随着你遇到新场景，继续这个观察-细化-测试循环。每次迭代基于真实的 agent 行为而非假设来改进技能。

**收集团队反馈：**

1. 与团队成员分享技能并观察他们的使用
2. 询问：技能是否按预期激活？指令是否清晰？缺少什么？
3. 整合反馈以解决你自己使用模式中的盲点

**为什么这种方法有效**：Agent A 理解 agent 需求，你提供领域专业知识，Agent B 通过真实使用揭示差距，迭代细化基于观察到的行为而非假设来改进技能。

### 观察 agent 如何导航技能

当你迭代技能时，注意 agent 在实践中如何使用它们。观察：

* **意外的探索路径**：agent 是否以你未预料到的顺序读取文件？这可能表明你的结构没有你想象的那么直观
* **遗漏的连接**：agent 是否未能跟随指向重要文件的引用？你的链接可能需要更明确或更突出
* **对某些部分的过度依赖**：如果 agent 反复读取同一文件，考虑该内容是否应该放在主 SKILL.md 中
* **被忽略的内容**：如果 agent 从未访问某个捆绑文件，它可能是不必要的或在主指令中没有被充分指示

基于这些观察而非假设进行迭代。技能元数据中的 name 和 description 尤其关键。Agent 在决定是否针对当前任务触发技能时使用这些。确保它们清楚地描述技能做什么以及何时应使用。

## 要避免的反模式

### 避免 Windows 风格的路径

始终在文件路径中使用正斜杠，即使在 Windows 上：

* Good: `scripts/helper.py`、`reference/guide.md`
* Avoid: `scripts\helper.py`、`reference\guide.md`

Unix 风格的路径在所有平台上都能工作，而 Windows 风格的路径在 Unix 系统上会导致错误。

### 避免提供太多选项

除非必要，不要呈现多种方法：

````markdown  theme={null}
**坏的示例：太多选择**（令人困惑）：
"You can use pypdf, or pdfplumber, or PyMuPDF, or pdf2image, or..."

**好的示例：提供默认值**（带逃生出口）：
"Use pdfplumber for text extraction:
```python
import pdfplumber
```

For scanned PDFs requiring OCR, use pdf2image with pytesseract instead."
````

## 高级：带有可执行代码的技能

以下部分聚焦于包含可执行脚本的技能。如果你的技能仅使用 markdown 指令，跳到[有效技能清单](#checklist-for-effective-skills)。

### 解决问题，而非推卸责任

在为技能编写脚本时，处理错误条件而不是推卸给 agent。

**好的示例：明确处理错误**：

```python  theme={null}
def process_file(path):
    """Process a file, creating it if it doesn't exist."""
    try:
        with open(path) as f:
            return f.read()
    except FileNotFoundError:
        # Create file with default content instead of failing
        print(f"File {path} not found, creating default")
        with open(path, 'w') as f:
            f.write('')
        return ''
    except PermissionError:
        # Provide alternative instead of failing
        print(f"Cannot access {path}, using default")
        return ''
```

**坏的示例：推卸给 agent**：

```python  theme={null}
def process_file(path):
    # Just fail and let the agent figure it out
    return open(path).read()
```

配置参数也应有理有据并加以文档化，以避免"巫术常量"（Ousterhout 定律）。如果你不知道正确的值，agent 将如何确定它？

**好的示例：自文档化**：

```python  theme={null}
# HTTP requests typically complete within 30 seconds
# Longer timeout accounts for slow connections
REQUEST_TIMEOUT = 30

# Three retries balances reliability vs speed
# Most intermittent failures resolve by the second retry
MAX_RETRIES = 3
```

**坏的示例：魔数**：

```python  theme={null}
TIMEOUT = 47  # Why 47?
RETRIES = 5   # Why 5?
```

### 提供实用脚本

即使你的 agent 可以编写脚本，预制的脚本也有优势：

**实用脚本的好处**：

* 比生成的代码更可靠
* 节省 token（无需在上下文中包含代码）
* 节省时间（无需代码生成）
* 确保跨使用的一致性

**重要区别**：在你的指令中明确说明 agent 应该：

* **执行脚本**（最常见）："Run `analyze_form.py` to extract fields"
* **作为参考阅读**（用于复杂逻辑）："See `analyze_form.py` for the field extraction algorithm"

对于大多数实用脚本，执行是首选，因为它更可靠、更高效。有关脚本执行如何工作的详细信息，请参见下文的运行时环境部分。

**示例**：

````markdown  theme={null}
## Utility scripts

**analyze_form.py**: Extract all form fields from PDF

```bash
python scripts/analyze_form.py input.pdf > fields.json
```

Output format:
```json
{
  "field_name": {"type": "text", "x": 100, "y": 200},
  "signature": {"type": "sig", "x": 150, "y": 500}
}
```

**validate_boxes.py**: Check for overlapping bounding boxes

```bash
python scripts/validate_boxes.py fields.json
# Returns: "OK" or lists conflicts
```

**fill_form.py**: Apply field values to PDF

```bash
python scripts/fill_form.py input.pdf fields.json output.pdf
```
````

### 使用视觉分析

当输入可以渲染为图像时，让 agent 进行分析：

````markdown  theme={null}
## Form layout analysis

1. Convert PDF to images:
   ```bash
   python scripts/pdf_to_images.py form.pdf
   ```

2. Analyze each page image to identify form fields
3. The agent can see field locations and types visually
````

<Note>
  在此示例中，你需要编写 `pdf_to_images.py` 脚本。
</Note>

Agent 的视觉能力有助于理解布局和结构。

### 创建可验证的中间输出

当 agent 执行复杂的开放式任务时，它们可能犯错误。"计划-验证-执行"模式通过让 agent 首先以结构化格式创建计划，然后在执行之前用脚本验证该计划来及早捕获错误。

**示例**：想象要求 agent 根据电子表格更新 PDF 中的 50 个表单字段。没有验证的话，它可能引用不存在的字段、创建冲突的值、遗漏必需字段或错误应用更新。

**解决方案**：使用上述工作流模式（PDF 表单填写），但添加一个中间的 `changes.json` 文件，在应用更改之前进行验证。工作流变为：分析 -> **创建计划文件** -> **验证计划** -> 执行 -> 验证。

**为什么此模式有效：**

* **及早捕获错误**：验证在更改应用之前发现问题
* **机器可验证**：脚本提供客观验证
* **可逆计划**：Agent 可以在不触及原始文件的情况下迭代计划
* **清晰的调试**：错误消息指向具体问题

**使用时机**：批量操作、破坏性更改、复杂验证规则、高风险操作。

**实施提示**：使验证脚本提供详细且具体的错误消息，如 "Field 'signature_date' not found. Available fields: customer_name, order_total, signature_date_signed"，以帮助 agent 修复问题。

### 打包依赖

技能在代码执行环境中运行，具有平台特定的限制：

* **claude.ai**：可以从 npm 和 PyPI 安装包，以及从 GitHub 仓库拉取
* **Anthropic API**：没有网络访问权限，也没有运行时包安装

在你的 SKILL.md 中列出所需的包，并在[代码执行工具文档](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool)中验证它们是否可用。

### 运行时环境

技能在具有文件系统访问、bash 命令和代码执行能力的代码执行环境中运行。关于此架构的概念性解释，请参见概述中的[技能架构](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#the-skills-architecture)。

**这如何影响你的编写：**

**Agent 如何访问技能：**

1. **元数据预加载**：启动时，所有技能的 YAML frontmatter 中的 name 和 description 被加载到系统提示词中
2. **文件按需读取**：Agent 在需要时使用其文件读取工具从文件系统访问 SKILL.md 和其他文件
3. **脚本高效执行**：实用脚本可以通过 bash 执行，而不会将其全部内容加载到上下文中。只有脚本的输出消耗 token
4. **大文件无上下文惩罚**：参考文件、数据或文档在实际读取之前不消耗上下文 token

* **文件路径很重要**：Agent 像浏览文件系统一样浏览你的技能目录。使用正斜杠（`reference/guide.md`），而非反斜杠
* **描述性命名文件**：使用指示内容的名称：`form_validation_rules.md`，而非 `doc2.md`
* **为发现而组织**：按领域或功能组织目录
  * 好：`reference/finance.md`、`reference/sales.md`
  * 坏：`docs/file1.md`、`docs/file2.md`
* **捆绑综合资源**：包含完整的 API 文档、大量示例、大数据集；在访问之前无上下文惩罚
* **对确定性操作优先使用脚本**：编写 `validate_form.py` 而非要求 agent 生成验证代码
* **明确执行意图**：
  * "Run `analyze_form.py` to extract fields"（执行）
  * "See `analyze_form.py` for the extraction algorithm"（作为参考阅读）
* **测试文件访问模式**：通过使用真实请求测试来验证 agent 能否导航你的目录结构

**示例：**

```
bigquery-skill/
├── SKILL.md (概述，指向参考文件)
└── reference/
    ├── finance.md (收入指标)
    ├── sales.md (pipeline 数据)
    └── product.md (使用分析)
```

当用户询问收入时，agent 读取 SKILL.md，看到指向 `reference/finance.md` 的引用，并调用 bash 只读取该文件。sales.md 和 product.md 文件保留在文件系统上，需要之前消耗零上下文 token。这种基于文件系统的模型正是实现渐进式披露的机制。Agent 可以导航并选择性地加载每个任务所需的确切内容。

关于技术架构的完整细节，请参见技能概述中的[技能如何工作](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#how-skills-work)。

### MCP 工具引用

如果你的技能使用 MCP（Model Context Protocol）工具，始终使用完全限定的工具名称以避免"工具未找到"错误。

**格式**：`ServerName:tool_name`

**示例**：

```markdown  theme={null}
Use the BigQuery:bigquery_schema tool to retrieve table schemas.
Use the GitHub:create_issue tool to create issues.
```

其中：

* `BigQuery` 和 `GitHub` 是 MCP 服务器名称
* `bigquery_schema` 和 `create_issue` 是这些服务器中的工具名称

如果没有服务器前缀，agent 可能无法定位工具，尤其是在多个 MCP 服务器可用时。

### 避免假设工具已安装

不要假设包可用：

````markdown  theme={null}
**坏的示例：假设已安装**：
"Use the pdf library to process the file."

**好的示例：明确说明依赖**：
"Install required package: `pip install pypdf`

Then use it:
```python
from pypdf import PdfReader
reader = PdfReader("file.pdf")
```"
````

## 技术说明

### YAML frontmatter 要求

SKILL.md 的 frontmatter 需要 `name`（最多 64 个字符）和 `description`（最多 1024 个字符）字段。完整的结构细节，请参见[技能概述](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#skill-structure)。

### Token 预算

保持 SKILL.md 正文在 500 行以内以获得最佳性能。如果你的内容超过此限制，使用前面描述的渐进式披露模式将其拆分到独立文件中。有关架构细节，请参见[技能概述](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#how-skills-work)。

## 有效技能清单

分享技能之前，验证：

### 核心质量

* [ ] 描述包含具体和关键术语
* [ ] 描述包含技能做什么以及何时使用
* [ ] SKILL.md 正文在 500 行以内
* [ ] 额外细节在独立文件中（如需要）
* [ ] 无时间敏感信息（或在"old patterns"部分）
* [ ] 全文术语一致
* [ ] 示例具体而非抽象
* [ ] 文件引用一层深度
* [ ] 适当使用了渐进式披露
* [ ] 工作流有清晰的步骤

### 代码和脚本

* [ ] 脚本解决问题而非推卸给 agent
* [ ] 错误处理明确且有帮助
* [ ] 无"巫术常量"（所有值都有理有据）
* [ ] 所需包在指令中列出并验证可用
* [ ] 脚本有清晰的文档
* [ ] 无 Windows 风格路径（全部为正斜杠）
* [ ] 关键操作有验证/校验步骤
* [ ] 质量关键任务包含反馈循环

### 测试

* [ ] 创建了至少三个评估
* [ ] 使用 Haiku、Sonnet 和 Opus 进行了测试
* [ ] 使用真实使用场景进行了测试
* [ ] 整合了团队反馈（如适用）

## 后续步骤

<CardGroup cols={2}>
  <Card title="Get started with Agent Skills" icon="rocket" href="https://platform.claude.com/docs/en/agents-and-tools/agent-skills/quickstart">
    Create your first Skill
  </Card>

  <Card title="Use Skills in Claude Code" icon="terminal" href="https://code.claude.com/docs/en/skills">
    Create and manage Skills in Claude Code
  </Card>

  <Card title="Use Skills with the API" icon="code" href="https://platform.claude.com/docs/en/build-with-claude/skills-guide">
    Upload and use Skills programmatically
  </Card>
</CardGroup>
