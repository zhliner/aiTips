# Skill 编写最佳实践

> 了解如何编写有效的 Skills，使 agents 能够成功发现和使用它们。

好的 Skills 简洁、结构良好，并经过真实使用测试。本指南提供实用的编写决策，帮助你编写 agents 能有效发现和使用的 Skills。

有关 Skills 工作原理的概念背景，请参阅 [Skills 概述](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)。

## 核心原则

### 简洁是关键

[上下文窗口](https://platform.claude.com/docs/en/build-with-claude/context-windows)是公共资源。你的 Skill 与 agent 需要知道的其他所有内容共享上下文窗口，包括：

* 系统提示
* 对话历史
* 其他 Skills 的元数据
* 你的实际请求

并非 Skill 中的每个 token 都有即时成本。在启动时，仅预加载所有 Skills 的元数据（name 和 description）。Agents 仅在 Skill 变得相关时才读取 SKILL.md，并且仅在需要时才读取附加文件。然而，在 SKILL.md 中保持简洁仍然很重要：一旦 agent 加载它，每个 token 都与对话历史和其他上下文竞争。

**默认假设**：Agents 已经很聪明

只添加 agents 尚不具备的上下文。质疑每条信息：

* "Agent 真的需要这个解释吗？"
* "我能假设 agent 已经知道这个吗？"
* "这段内容值得它的 token 成本吗？"

**好的示例：简洁**（约 50 个 token）：

````markdown  theme={null}
## 提取 PDF 文本

使用 pdfplumber 提取文本：

```python
import pdfplumber

with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```
````

**不好的示例：过于冗长**（约 150 个 token）：

```markdown  theme={null}
## 提取 PDF 文本

PDF（便携式文档格式）文件是一种常见的文件格式，包含
文本、图像和其他内容。要从 PDF 中提取文本，你需要
使用一个库。有许多可用于 PDF 处理的库，但我们
推荐 pdfplumber，因为它易于使用且能处理大多数情况。
首先，你需要使用 pip 安装它。然后你可以使用下面的代码...
```

简洁版本假设 agent 知道什么是 PDF 以及库是如何工作的。

### 设定适当的自由度

将具体程度与任务的脆弱性和可变性匹配。

**高自由度**（基于文本的指令）：

适用场景：

* 多种方法都有效
* 决策取决于上下文
* 启发式方法指导方法

示例：

```markdown  theme={null}
## 代码审查流程

1. 分析代码结构和组织
2. 检查潜在的 bug 或边界情况
3. 建议提高可读性和可维护性的改进
4. 验证是否遵循项目约定
```

**中等自由度**（伪代码或带参数的脚本）：

适用场景：

* 存在首选模式
* 允许一定变化
* 配置影响行为

示例：

````markdown  theme={null}
## 生成报告

使用此模板并根据需要定制：

```python
def generate_report(data, format="markdown", include_charts=True):
    # 处理数据
    # 按指定格式生成输出
    # 可选包含可视化图表
```
````

**低自由度**（特定脚本，很少或没有参数）：

适用场景：

* 操作脆弱且容易出错
* 一致性至关重要
* 必须遵循特定顺序

示例：

````markdown  theme={null}
## 数据库迁移

严格运行此脚本：

```bash
python scripts/migrate.py --verify --backup
```

不要修改命令或添加额外的标志。
````

**类比**：将 agent 想象成在路径上探索的机器人：

* **两侧是悬崖的窄桥**：只有一条安全的前进路径。提供具体的护栏和精确的指令（低自由度）。示例：必须按精确顺序运行的数据库迁移。
* **没有危险的开阔场地**：许多路径都能通向成功。给出大致方向并信任 agent 能找到最佳路线（高自由度）。示例：上下文决定最佳方法的代码审查。

### 用你计划使用的所有模型进行测试

Skills 作为模型的附加组件，因此有效性取决于底层模型。用你计划使用的所有模型测试你的 Skill。

**按模型的测试注意事项**：

* **Claude Haiku**（快速、经济）：Skill 是否提供了足够的指导？
* **Claude Sonnet**（平衡）：Skill 是否清晰高效？
* **Claude Opus**（强大推理）：Skill 是否避免了过度解释？

对 Opus 完美有效的东西可能需要为 Haiku 提供更多细节。如果你计划在多个模型上使用 Skill，目标是编写对所有模型都效果良好的指令。

## Skill 结构

<Note>
  **YAML Frontmatter**：SKILL.md 的 frontmatter 需要两个字段：

  * `name` - Skill 的可读名称（最多 64 个字符）
  * `description` - Skill 做什么以及何时使用的一句话描述（最多 1024 个字符）

  有关完整的 Skill 结构详情，请参阅 [Skills 概述](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#skill-structure)。
</Note>

### 命名约定

使用一致的命名模式使 Skills 更容易引用和讨论。我们建议使用**动名词形式**（动词 + -ing）作为 Skill 名称，因为这清楚地描述了 Skill 提供的活动或能力。

**好的命名示例（动名词形式）**：

* "Processing PDFs"
* "Analyzing spreadsheets"
* "Managing databases"
* "Testing code"
* "Writing documentation"

**可接受的替代方案**：

* 名词短语："PDF Processing"、"Spreadsheet Analysis"
* 动作导向："Process PDFs"、"Analyze Spreadsheets"

**避免**：

* 模糊的名称："Helper"、"Utils"、"Tools"
* 过于通用："Documents"、"Data"、"Files"
* 在你的 skill 集合中不一致的模式

一致的命名使得更容易：

* 在文档和对话中引用 Skills
* 一目了然地理解 Skill 做什么
* 组织和搜索多个 Skills
* 维护一个专业、统一的 skill 库

### 编写有效的 descriptions

`description` 字段用于 Skill 发现，应该包含 Skill 做什么以及何时使用它。

<Warning>
  **始终使用第三人称撰写**。Description 会被注入到系统提示中，不一致的人称视角可能导致发现问题。

  * **好：** "Processes Excel files and generates reports"
  * **避免：** "I can help you process Excel files"
  * **避免：** "You can use this to process Excel files"
</Warning>

**要具体并包含关键术语**。包含 Skill 做什么以及使用它的具体触发器/上下文。

每个 Skill 只有一个 description 字段。Description 对于 skill 选择至关重要：agents 使用它从可能 100+ 个可用 Skills 中选择正确的 Skill。你的 description 必须提供足够的细节让 agent 知道何时选择此 Skill，而 SKILL.md 的其余部分提供实现细节。

有效示例：

**PDF 处理 skill：**

```yaml  theme={null}
description: Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
```

**Excel 分析 skill：**

```yaml  theme={null}
description: Analyze Excel spreadsheets, create pivot tables, generate charts. Use when analyzing Excel files, spreadsheets, tabular data, or .xlsx files.
```

**Git Commit 助手 skill：**

```yaml  theme={null}
description: Generate descriptive commit messages by analyzing git diffs. Use when the user asks for help writing commit messages or reviewing staged changes.
```

避免这样的模糊描述：

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

SKILL.md 作为概述，在需要时引导 agents 查看详细材料，就像入门指南中的目录。有关渐进式披露工作原理的解释，请参阅概述中的 [Skills 的工作原理](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#how-skills-work)。

**实用指导：**

* 将 SKILL.md 正文保持在 500 行以内以获得最佳性能
* 接近此限制时将内容拆分到单独文件中
* 使用下面的模式有效组织指令、代码和资源

#### 视觉概述：从简单到复杂

一个基本的 Skill 从一个包含元数据和指令的 SKILL.md 文件开始：

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=87782ff239b297d9a9e8e1b72ed72db9" alt="显示 YAML frontmatter 和 markdown 正文的简单 SKILL.md 文件" data-og-width="2048" width="2048" data-og-height="1153" height="1153" data-path="images/agent-skills-simple-file.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=c61cc33b6f5855809907f7fda94cd80e 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=90d2c0c1c76b36e8d485f49e0810dbfd 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=ad17d231ac7b0bea7e5b4d58fb4aeabb 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=f5d0a7a3c668435bb0aee9a3a8f8c329 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=0e927c1af9de5799cfe557d12249f6e6 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=46bbb1a51dd4c8202a470ac8c80a893d 2500w" />

随着 Skill 的增长，你可以捆绑额外的内容，agents 仅在需要时加载：

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=a5e0aa41e3d53985a7e3e43668a33ea3" alt="捆绑额外的参考文件，如 reference.md 和 forms.md。" data-og-width="2048" width="2048" data-og-height="1327" height="1327" data-path="images/agent-skills-bundling-content.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=f8a0e73783e99b4a643d79eac86b70a2 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=dc510a2a9d3f14359416b706f067904a 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=82cd6286c966303f7dd914c28170e385 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=56f3be36c77e4fe4b523df209a6824c6 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=d22b5161b2075656417d56f41a74f3dd 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=3dd4bdd6850ffcc96c6c45fcb0acd6eb 2500w" />

完整的 Skill 目录结构可能如下所示：

```
pdf/
├── SKILL.md              # 主指令（触发时加载）
├── FORMS.md              # 表单填写指南（按需加载）
├── reference.md          # API 参考（按需加载）
├── examples.md           # 使用示例（按需加载）
└── scripts/
    ├── analyze_form.py   # 实用脚本（执行，不加载）
    ├── fill_form.py      # 表单填写脚本
    └── validate.py       # 验证脚本
```

#### 模式 1：带引用的概要指南

````markdown  theme={null}
---
name: PDF Processing
description: Extracts text and tables from PDF files, fills forms, and merges documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
---

# PDF 处理

## 快速开始

使用 pdfplumber 提取文本：
```python
import pdfplumber
with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```

## 高级功能

**表单填写**：完整指南参见 [FORMS.md](FORMS.md)
**API 参考**：所有方法参见 [REFERENCE.md](REFERENCE.md)
**示例**：常见模式参见 [EXAMPLES.md](EXAMPLES.md)
````

Agents 仅在需要时加载 FORMS.md、REFERENCE.md 或 EXAMPLES.md。

#### 模式 2：按领域组织

对于包含多个领域的 Skills，按领域组织内容以避免加载无关上下文。当用户询问销售指标时，agent 只需要读取与销售相关的 schema，而不是财务或营销数据。这保持了较低的 token 使用量和聚焦的上下文。

```
bigquery-skill/
├── SKILL.md (概述和导航)
└── reference/
    ├── finance.md (收入、计费指标)
    ├── sales.md (商机、销售管线)
    ├── product.md (API 使用、功能)
    └── marketing.md (营销活动、归因)
```

````markdown SKILL.md theme={null}
# BigQuery 数据分析

## 可用数据集

**财务**：收入、ARR、计费 → 参见 [reference/finance.md](reference/finance.md)
**销售**：商机、管线、客户 → 参见 [reference/sales.md](reference/sales.md)
**产品**：API 使用、功能、采用率 → 参见 [reference/product.md](reference/product.md)
**营销**：活动、归因、邮件 → 参见 [reference/marketing.md](reference/marketing.md)

## 快速搜索

使用 grep 查找特定指标：

```bash
grep -i "revenue" reference/finance.md
grep -i "pipeline" reference/sales.md
grep -i "api usage" reference/product.md
```
````

#### 模式 3：条件详情

展示基础内容，链接到高级内容：

```markdown  theme={null}
# DOCX 处理

## 创建文档

使用 docx-js 创建新文档。参见 [DOCX-JS.md](DOCX-JS.md)。

## 编辑文档

对于简单编辑，直接修改 XML。

**修订跟踪**：参见 [REDLINING.md](REDLINING.md)
**OOXML 详情**：参见 [OOXML.md](OOXML.md)
```

Agents 仅在用户需要这些功能时才读取 REDLINING.md 或 OOXML.md。

### 避免深层嵌套引用

Agents 在从其他引用文件引用的文件中可能只会部分读取。当遇到嵌套引用时，agent 可能使用 `head -100` 等命令预览内容而不是读取完整文件，导致信息不完整。

**保持引用在 SKILL.md 的一层深度内**。所有参考文件应从 SKILL.md 直接链接，以确保 agents 在需要时读取完整文件。

**不好的示例：嵌套太深**：

```markdown  theme={null}
# SKILL.md
参见 [advanced.md](advanced.md)...

# advanced.md
参见 [details.md](details.md)...

# details.md
这里是实际的信息...
```

**好的示例：一层深度**：

```markdown  theme={null}
# SKILL.md

**基本用法**：[SKILL.md 中的说明]
**高级功能**：参见 [advanced.md](advanced.md)
**API 参考**：参见 [reference.md](reference.md)
**示例**：参见 [examples.md](examples.md)
```

### 用目录结构化较长的参考文件

对于超过 100 行的参考文件，在顶部包含目录。这确保 agents 即使在使用部分读取预览时也能看到可用信息的完整范围。

**示例**：

```markdown  theme={null}
# API 参考

## 目录
- 认证与设置
- 核心方法（创建、读取、更新、删除）
- 高级功能（批量操作、webhooks）
- 错误处理模式
- 代码示例

## 认证与设置
...

## 核心方法
...
```

Agents 然后可以读取完整文件或在需要时跳转到特定部分。

有关此基于文件系统的架构如何实现渐进式披露的详情，请参阅下面高级部分中的[运行时环境](#运行时环境)。

## 工作流和反馈循环

### 对复杂任务使用工作流

将复杂操作分解为清晰的顺序步骤。对于特别复杂的工作流，提供一个清单，agent 可以复制到其响应中并在进展时勾选。

**示例 1：研究综合工作流**（适用于不含代码的 Skills）：

````markdown  theme={null}
## 研究综合工作流

复制此检查清单并跟踪你的进展：

```
研究进展：
- [ ] 步骤 1：阅读所有源文档
- [ ] 步骤 2：识别关键主题
- [ ] 步骤 3：交叉引用声明
- [ ] 步骤 4：创建结构化摘要
- [ ] 步骤 5：验证引用
```

**步骤 1：阅读所有源文档**

审阅 `sources/` 目录中的每个文档。记录主要论点和支撑证据。

**步骤 2：识别关键主题**

寻找跨来源的模式。哪些主题反复出现？哪些来源一致或存在分歧？

**步骤 3：交叉引用声明**

对于每个主要声明，验证它是否出现在源材料中。记录哪个来源支持每个观点。

**步骤 4：创建结构化摘要**

按主题组织发现。包括：
- 主要声明
- 来自来源的支撑证据
- 冲突的观点（如有）

**步骤 5：验证引用**

检查每个声明是否引用了正确的源文档。如果引用不完整，返回步骤 3。
````

此示例展示了工作流如何应用于不需要代码的分析任务。清单模式适用于任何复杂的多步骤流程。

**示例 2：PDF 表单填写工作流**（适用于含代码的 Skills）：

````markdown  theme={null}
## PDF 表单填写工作流

复制此检查清单并在完成时勾选：

```
任务进展：
- [ ] 步骤 1：分析表单（运行 analyze_form.py）
- [ ] 步骤 2：创建字段映射（编辑 fields.json）
- [ ] 步骤 3：验证映射（运行 validate_fields.py）
- [ ] 步骤 4：填写表单（运行 fill_form.py）
- [ ] 步骤 5：验证输出（运行 verify_output.py）
```

**步骤 1：分析表单**

运行：`python scripts/analyze_form.py input.pdf`

这会提取表单字段及其位置，保存到 `fields.json`。

**步骤 2：创建字段映射**

编辑 `fields.json` 为每个字段添加值。

**步骤 3：验证映射**

运行：`python scripts/validate_fields.py fields.json`

在继续之前修复任何验证错误。

**步骤 4：填写表单**

运行：`python scripts/fill_form.py input.pdf fields.json output.pdf`

**步骤 5：验证输出**

运行：`python scripts/verify_output.py output.pdf`

如果验证失败，返回步骤 2。
````

清晰的步骤防止 agents 跳过关键验证。清单帮助你和 agent 跟踪多步骤工作流的进展。

### 实现反馈循环

**常见模式**：运行验证器 → 修复错误 → 重复

这种模式极大地提高了输出质量。

**示例 1：风格指南合规**（适用于不含代码的 Skills）：

```markdown  theme={null}
## 内容审查流程

1. 按照 STYLE_GUIDE.md 中的指南起草内容
2. 对照检查清单审查：
   - 检查术语一致性
   - 验证示例是否遵循标准格式
   - 确认所有必需部分都存在
3. 如果发现问题：
   - 记录每个问题并引用具体章节
   - 修改内容
   - 再次审查检查清单
4. 只有满足所有要求后才继续
5. 最终确定并保存文档
```

这展示了使用参考文档而非脚本的验证循环模式。"验证器"是 STYLE\_GUIDE.md，agent 通过阅读和比较来执行检查。

**示例 2：文档编辑流程**（适用于含代码的 Skills）：

```markdown  theme={null}
## 文档编辑流程

1. 对 `word/document.xml` 进行编辑
2. **立即验证**：`python ooxml/scripts/validate.py unpacked_dir/`
3. 如果验证失败：
   - 仔细审查错误消息
   - 修复 XML 中的问题
   - 再次运行验证
4. **只有在验证通过后才继续**
5. 重新打包：`python ooxml/scripts/pack.py unpacked_dir/ output.docx`
6. 测试输出文档
```

验证循环能及早捕获错误。

## 内容指南

### 避免时效性信息

不要包含会过时的信息：

**不好的示例：时效性**（将会变得不正确）：

```markdown  theme={null}
如果你在 2025 年 8 月之前执行此操作，使用旧 API。
2025 年 8 月之后，使用新 API。
```

**好的示例**（使用"旧模式"部分）：

```markdown  theme={null}
## 当前方法

使用 v2 API 端点：`api.example.com/v2/messages`

## 旧模式

<details>
<summary>旧版 v1 API（2025-08 已弃用）</summary>

v1 API 使用：`api.example.com/v1/messages`

此端点不再受支持。
</details>
```

旧模式部分提供了历史上下文而不会使主内容变得杂乱。

### 使用一致的术语

选择一个术语并在整个 Skill 中保持一致使用：

**好的 - 一致**：

* 始终使用 "API endpoint"
* 始终使用 "field"
* 始终使用 "extract"

**不好的 - 不一致**：

* 混用 "API endpoint"、"URL"、"API route"、"path"
* 混用 "field"、"box"、"element"、"control"
* 混用 "extract"、"pull"、"get"、"retrieve"

一致性帮助 agents 理解和遵循指令。

## 常见模式

### 模板模式

为输出格式提供模板。根据你的需求匹配严格程度。

**对于严格要求**（如 API 响应或数据格式）：

````markdown  theme={null}
## 报告结构

始终使用此精确模板结构：

```markdown
# [分析标题]

## 执行摘要
[关键发现的一段概述]

## 关键发现
- 发现 1 及支撑数据
- 发现 2 及支撑数据
- 发现 3 及支撑数据

## 建议
1. 具体可操作的建议
2. 具体可操作的建议
```
````

**对于灵活指导**（当适配有用时）：

````markdown  theme={null}
## 报告结构

这是一个合理的默认格式，但请根据分析使用你的最佳判断：

```markdown
# [分析标题]

## 执行摘要
[概述]

## 关键发现
[根据发现调整章节]

## 建议
[根据具体情境定制]
```

根据具体分析类型按需调整章节。
````

### 示例模式

对于输出质量取决于查看示例的 Skills，提供输入/输出对，就像常规 prompting 一样：

````markdown  theme={null}
## 提交消息格式

按照以下示例生成提交消息：

**示例 1：**
输入：Added user authentication with JWT tokens
输出：
```
feat(auth): implement JWT-based authentication

Add login endpoint and token validation middleware
```

**示例 2：**
输入：Fixed bug where dates displayed incorrectly in reports
输出：
```
fix(reports): correct date formatting in timezone conversion

Use UTC timestamps consistently across report generation
```

**示例 3：**
输入：Updated dependencies and refactored error handling
输出：
```
chore: update dependencies and refactor error handling

- Upgrade lodash to 4.17.21
- Standardize error response format across endpoints
```

遵循此风格：type(scope): 简要描述，然后是详细说明。
````

示例帮助 agents 比仅靠描述更清楚地理解期望的风格和细节程度。

### 条件工作流模式

引导 agents 通过决策点：

```markdown  theme={null}
## 文档修改工作流

1. 确定修改类型：

   **创建新内容？** → 遵循下面的"创建工作流"
   **编辑现有内容？** → 遵循下面的"编辑工作流"

2. 创建工作流：
   - 使用 docx-js 库
   - 从头构建文档
   - 导出为 .docx 格式

3. 编辑工作流：
   - 解包现有文档
   - 直接修改 XML
   - 每次更改后验证
   - 完成后重新打包
```

<Tip>
  如果工作流变得庞大或复杂且步骤繁多，考虑将它们推送到单独文件中，并告诉 agent 根据当前任务读取相应的文件。
</Tip>

## 评估和迭代

### 先构建评估

**在编写大量文档之前先创建评估。** 这确保你的 Skill 解决真实问题而非记录想象中的问题。

**评估驱动开发：**

1. **识别空白**：在没有 Skill 的情况下让 agent 执行代表性任务。记录具体的失败或缺失的上下文
2. **创建评估**：构建三个测试这些空白的场景
3. **建立基线**：衡量 agent 在没有 Skill 时的表现
4. **编写最少指令**：创建刚好足够的内容来解决空白并通过评估
5. **迭代**：执行评估，与基线比较，并改进

这种方法确保你解决的是实际问题，而非预期可能永远不会出现的需求。

**评估结构**：

```json  theme={null}
{
  "skills": ["pdf-processing"],
  "query": "Extract all text from this PDF file and save it to output.txt",
  "files": ["test-files/document.pdf"],
  "expected_behavior": [
    "使用合适的 PDF 处理库或命令行工具成功读取 PDF 文件",
    "从文档所有页面提取文本内容，不遗漏任何页面",
    "将提取的文本以清晰可读的格式保存到名为 output.txt 的文件中"
  ]
}
```

<Note>
  此示例展示了一个数据驱动的评估，带有简单的测试评分标准。我们目前没有提供内置的方式来运行这些评估。用户可以创建自己的评估系统。评估是你衡量 Skill 有效性的真实来源。
</Note>

### 与 agent 迭代开发 Skills

最有效的 Skill 开发流程涉及 agent 本身。与一个实例（"Agent A"）合作创建一个将被其他实例（"Agent B"）使用的 Skill。Agent A 帮助你设计和改进指令，而 Agent B 在真实任务中测试它们。这之所以有效，是因为底层模型理解如何编写有效的 agent 指令以及 agents 需要什么信息。

**创建新 Skill：**

1. **在没有 Skill 的情况下完成任务**：使用常规 prompting 与 Agent A 一起解决问题。在工作过程中，你自然会提供上下文、解释偏好并分享程序性知识。注意你反复提供的信息。

2. **识别可复用模式**：完成任务后，识别你提供的哪些上下文对未来类似任务有用。

   **示例**：如果你完成了 BigQuery 分析，你可能提供了表名、字段定义、过滤规则（如"始终排除测试账户"）和常见查询模式。

3. **让 Agent A 创建 Skill**："创建一个 Skill 来捕获我们刚才使用的这个 BigQuery 分析模式。包含表 schema、命名约定以及过滤测试账户的规则。"

   <Tip>
     现代 agents 原生理解 Skill 的格式和结构。你不需要特殊的系统提示或"编写 skills" skill 来获取创建 Skills 的帮助。直接让 agent 创建 Skill，它就会生成结构正确的 SKILL.md 内容，包含适当的 frontmatter 和正文。
   </Tip>

4. **检查简洁性**：确认 Agent A 没有添加不必要的解释。问："删除关于胜率含义的解释——agent 已经知道了。"

5. **改进信息架构**：让 Agent A 更有效地组织内容。例如："组织这些内容，使表 schema 放在单独的参考文件中。我们以后可能会添加更多表。"

6. **在类似任务上测试**：在相关用例中使用 Agent B（加载了 Skill 的新实例）测试 Skill。观察 Agent B 是否找到了正确的信息、正确应用了规则并成功处理了任务。

7. **基于观察迭代**：如果 Agent B 遇到困难或遗漏了什么，带着具体情况回到 Agent A："当 agent 使用这个 Skill 时，它忘记了按日期过滤 Q4。我们应该添加一个关于日期过滤模式的部分吗？"

**迭代改进现有 Skills：**

在改进 Skills 时，同样的层级模式继续适用。你在以下之间交替：

* **与 Agent A 合作**（帮助改进 Skill 的专家）
* **用 Agent B 测试**（使用 Skill 执行实际工作的 agent）
* **观察 Agent B 的行为**并将洞察带回给 Agent A

1. **在真实工作流中使用 Skill**：给 Agent B（加载了 Skill）实际任务，而非测试场景

2. **观察 Agent B 的行为**：注意它在哪里遇到困难、成功或做出意外选择

   **观察示例**："当我让 Agent B 生成区域销售报告时，它编写了查询但忘记了过滤掉测试账户，即使 Skill 中提到了这条规则。"

3. **回到 Agent A 寻求改进**：分享当前的 SKILL.md 并描述你观察到的情况。问："我注意到 Agent B 在我要区域报告时忘记了过滤测试账户。Skill 提到了过滤，但可能不够突出？"

4. **审查 Agent A 的建议**：Agent A 可能会建议重新组织以使规则更突出，使用更强的语言如 "MUST filter" 而非 "always filter"，或重构工作流部分。

5. **应用并测试更改**：用 Agent A 的改进更新 Skill，然后在类似请求上再次用 Agent B 测试

6. **基于使用重复**：在遇到新场景时继续这个观察-改进-测试循环。每次迭代都基于真实的 agent 行为而非假设来改进 Skill。

**收集团队反馈：**

1. 与队友分享 Skills 并观察他们的使用
2. 询问：Skill 是否在预期时激活？指令是否清晰？缺少什么？
3. 纳入反馈以解决你自己使用模式中的盲点

**为什么这种方法有效**：Agent A 理解 agent 需求，你提供领域专业知识，Agent B 通过真实使用揭示空白，迭代改进基于观察到的行为而非假设来改进 Skills。

### 观察 agents 如何导航 Skills

在迭代 Skills 时，注意 agents 实际如何使用它们。观察：

* **意外的探索路径**：Agent 是否按你没预料到的顺序读取文件？这可能表明你的结构不如你想象的直观
* **遗漏的连接**：Agent 是否没有跟随对重要文件的引用？你的链接可能需要更明确或更突出
* **过度依赖某些部分**：如果 agent 反复读取同一个文件，考虑该内容是否应该在主 SKILL.md 中
* **被忽略的内容**：如果 agent 从不访问捆绑的文件，它可能是不必要的或在主指令中信号不佳

基于这些观察而非假设进行迭代。Skill 元数据中的 'name' 和 'description' 特别关键。Agents 在决定是否响应当前任务触发 Skill 时使用它们。确保它们清楚地描述了 Skill 做什么以及何时应该使用它。

## 应避免的反模式

### 避免 Windows 风格路径

在文件路径中始终使用正斜杠，即使在 Windows 上：

* ✓ **好**：`scripts/helper.py`、`reference/guide.md`
* ✗ **避免**：`scripts\helper.py`、`reference\guide.md`

Unix 风格路径在所有平台上工作，而 Windows 风格路径在 Unix 系统上会导致错误。

### 避免提供过多选项

不要呈现多种方法，除非必要：

````markdown  theme={null}
**不好的示例：选择太多**（令人困惑）：
"你可以使用 pypdf，或 pdfplumber，或 PyMuPDF，或 pdf2image，或..."

**好的示例：提供默认选项**（带备选方案）：
"使用 pdfplumber 提取文本：
```python
import pdfplumber
```

对于需要 OCR 的扫描 PDF，改用 pdf2image 配合 pytesseract。"
````

## 高级：包含可执行代码的 Skills

以下部分侧重于包含可执行脚本的 Skills。如果你的 Skill 仅使用 markdown 指令，请跳到[有效 Skills 清单](#有效-skills-清单)。

### 解决问题，而非推卸

在为 Skills 编写脚本时，处理错误条件而非推给 agent。

**好的示例：明确处理错误**：

```python  theme={null}
def process_file(path):
    """处理文件，如果不存在则创建。"""
    try:
        with open(path) as f:
            return f.read()
    except FileNotFoundError:
        # 创建默认内容的文件而非失败
        print(f"File {path} not found, creating default")
        with open(path, 'w') as f:
            f.write('')
        return ''
    except PermissionError:
        # 使用替代方案而非失败
        print(f"Cannot access {path}, using default")
        return ''
```

**不好的示例：推给 agent**：

```python  theme={null}
def process_file(path):
    # 直接失败，让 agent 自己处理
    return open(path).read()
```

配置参数也应该有合理的理由和文档记录，以避免"巫术常量"（Ousterhout 定律）。如果你不知道正确的值，agent 如何确定它？

**好的示例：自文档化**：

```python  theme={null}
# HTTP 请求通常在 30 秒内完成
# 更长的超时考虑了慢速连接
REQUEST_TIMEOUT = 30

# 三次重试在可靠性和速度之间取得平衡
# 大多数间歇性故障在第二次重试时解决
MAX_RETRIES = 3
```

**不好的示例：魔法数字**：

```python  theme={null}
TIMEOUT = 47  # 为什么是 47？
RETRIES = 5   # 为什么是 5？
```

### 提供实用脚本

即使你的 agent 可以编写脚本，预制脚本也有优势：

**实用脚本的好处**：

* 比生成的代码更可靠
* 节省 token（不需要在上下文中包含代码）
* 节省时间（不需要代码生成）
* 确保跨使用的一致性

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=4bbc45f2c2e0bee9f2f0d5da669bad00" alt="将可执行脚本与指令文件捆绑" data-og-width="2048" width="2048" data-og-height="1154" height="1154" data-path="images/agent-skills-executable-scripts.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=9a04e6535a8467bfeea492e517de389f 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=e49333ad90141af17c0d7651cca7216b 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=954265a5df52223d6572b6214168c428 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=2ff7a2d8f2a83ee8af132b29f10150fd 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=48ab96245e04077f4d15e9170e081cfb 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=0301a6c8b3ee879497cc5b5483177c90 2500w" />

上图展示了可执行脚本如何与指令文件协同工作。指令文件（forms.md）引用脚本，agent 可以执行它而无需将其内容加载到上下文中。

**重要区别**：在你的指令中明确说明 agent 应该：

* **执行脚本**（最常见）："Run `analyze_form.py` to extract fields"
* **作为参考阅读**（用于复杂逻辑）："See `analyze_form.py` for the field extraction algorithm"

对于大多数实用脚本，执行是首选，因为它更可靠和高效。有关脚本执行如何工作的详情，请参阅下面的[运行时环境](#运行时环境)部分。

**示例**：

````markdown  theme={null}
## 实用脚本

**analyze_form.py**：从 PDF 提取所有表单字段

```bash
python scripts/analyze_form.py input.pdf > fields.json
```

输出格式：
```json
{
  "field_name": {"type": "text", "x": 100, "y": 200},
  "signature": {"type": "sig", "x": 150, "y": 500}
}
```

**validate_boxes.py**：检查边界框是否重叠

```bash
python scripts/validate_boxes.py fields.json
# 返回："OK" 或列出冲突
```

**fill_form.py**：将字段值应用到 PDF

```bash
python scripts/fill_form.py input.pdf fields.json output.pdf
```
````

### 使用视觉分析

当输入可以渲染为图像时，让 agent 分析它们：

````markdown  theme={null}
## 表单布局分析

1. 将 PDF 转换为图像：
   ```bash
   python scripts/pdf_to_images.py form.pdf
   ```

2. 分析每个页面图像以识别表单字段
3. Agent 可以直观地看到字段位置和类型
````

<Note>
  在此示例中，你需要编写 `pdf_to_images.py` 脚本。
</Note>

Agent 的视觉能力有助于理解布局和结构。

### 创建可验证的中间输出

当 agents 执行复杂的开放式任务时，它们可能会犯错。"计划-验证-执行"模式通过让 agent 首先创建结构化格式的计划，然后在执行之前用脚本验证该计划来及早捕获错误。

**示例**：想象你让 agent 基于电子表格更新 PDF 中的 50 个表单字段。没有验证的情况下，它可能会引用不存在的字段、创建冲突的值、遗漏必填字段或不正确地应用更新。

**解决方案**：使用上面展示的工作流模式（PDF 表单填写），但添加一个在应用更改之前被验证的中间 `changes.json` 文件。工作流变为：分析 → **创建计划文件** → **验证计划** → 执行 → 验证。

**为什么这种模式有效：**

* **及早捕获错误**：验证在应用更改之前发现问题
* **机器可验证**：脚本提供客观验证
* **可逆的计划**：Agent 可以在不触及原始文件的情况下迭代计划
* **清晰的调试**：错误消息指向具体问题

**使用时机**：批量操作、破坏性更改、复杂验证规则、高风险操作。

**实现提示**：使验证脚本输出详细的具体错误消息，如 "Field 'signature\_date' not found. Available fields: customer\_name, order\_total, signature\_date\_signed"，以帮助 agent 修复问题。

### 包依赖

Skills 在具有平台特定限制的代码执行环境中运行：

* **claude.ai**：可以从 npm 和 PyPI 安装包并从 GitHub 仓库拉取
* **Anthropic API**：没有网络访问权限，也没有运行时包安装

在 SKILL.md 中列出所需的包，并在[代码执行工具文档](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool)中验证它们的可用性。

### 运行时环境

Skills 在具有文件系统访问、bash 命令和代码执行能力的代码执行环境中运行。有关此架构的概念性解释，请参阅概述中的 [Skills 架构](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#the-skills-architecture)。

**这对你的编写有何影响：**

**Agents 如何访问 Skills：**

1. **元数据预加载**：在启动时，所有 Skills 的 YAML frontmatter 中的 name 和 description 被加载到系统提示中
2. **文件按需读取**：Agents 使用其文件读取工具在需要时从文件系统访问 SKILL.md 和其他文件
3. **脚本高效执行**：实用脚本可以通过 bash 执行，而无需将其完整内容加载到上下文中。只有脚本的输出消耗 token
4. **大文件无上下文惩罚**：参考文件、数据或文档在实际读取之前不消耗上下文 token

* **文件路径很重要**：Agents 像文件系统一样导航你的 skill 目录。使用正斜杠（`reference/guide.md`），不使用反斜杠
* **描述性地命名文件**：使用指示内容的名称：`form_validation_rules.md`，而非 `doc2.md`
* **为发现而组织**：按领域或功能构建目录
  * 好：`reference/finance.md`、`reference/sales.md`
  * 不好：`docs/file1.md`、`docs/file2.md`
* **捆绑全面的资源**：包含完整的 API 文档、大量示例、大型数据集；在被访问之前没有上下文惩罚
* **对确定性操作优先使用脚本**：编写 `validate_form.py` 而非让 agent 生成验证代码
* **明确执行意图**：
  * "Run `analyze_form.py` to extract fields"（执行）
  * "See `analyze_form.py` for the extraction algorithm"（作为参考阅读）
* **测试文件访问模式**：通过真实请求测试来验证 agent 能否导航你的目录结构

**示例：**

```
bigquery-skill/
├── SKILL.md (概述，指向参考文件)
└── reference/
    ├── finance.md (收入指标)
    ├── sales.md (管线数据)
    └── product.md (使用分析)
```

当用户询问收入时，agent 读取 SKILL.md，看到对 `reference/finance.md` 的引用，然后调用 bash 仅读取该文件。sales.md 和 product.md 文件保留在文件系统上，在需要之前消耗零上下文 token。这种基于文件系统的模型实现了渐进式披露。Agents 可以导航并选择性地加载每个任务所需的内容。

有关技术架构的完整详情，请参阅 Skills 概述中的 [Skills 的工作原理](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#how-skills-work)。

### MCP 工具引用

如果你的 Skill 使用 MCP（Model Context Protocol）工具，始终使用完全限定的工具名称以避免"tool not found"错误。

**格式**：`ServerName:tool_name`

**示例**：

```markdown  theme={null}
使用 BigQuery:bigquery_schema 工具获取表 schema。
使用 GitHub:create_issue 工具创建 issue。
```

其中：

* `BigQuery` 和 `GitHub` 是 MCP 服务器名称
* `bigquery_schema` 和 `create_issue` 是这些服务器中的工具名称

没有服务器前缀，agents 可能无法定位工具，特别是当有多个 MCP 服务器可用时。

### 避免假设工具已安装

不要假设包已可用：

````markdown  theme={null}
**不好的示例：假设已安装**：
"使用 pdf 库处理文件。"

**好的示例：明确依赖项**：
"安装所需包：`pip install pypdf`

然后使用它：
```python
from pypdf import PdfReader
reader = PdfReader("file.pdf")
```"
````

## 技术说明

### YAML frontmatter 要求

SKILL.md 的 frontmatter 需要 `name`（最多 64 个字符）和 `description`（最多 1024 个字符）字段。有关完整的结构详情，请参阅 [Skills 概述](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#skill-structure)。

### Token 预算

将 SKILL.md 正文保持在 500 行以内以获得最佳性能。如果你的内容超过此限制，使用前面描述的渐进式披露模式将其拆分到单独文件中。有关架构详情，请参阅 [Skills 概述](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#how-skills-work)。

## 有效 Skills 清单

在分享 Skill 之前，验证：

### 核心质量

* [ ] Description 具体且包含关键术语
* [ ] Description 包含 Skill 做什么以及何时使用它
* [ ] SKILL.md 正文在 500 行以内
* [ ] 额外细节在单独文件中（如果需要）
* [ ] 没有时效性信息（或在"旧模式"部分中）
* [ ] 全文术语一致
* [ ] 示例具体而非抽象
* [ ] 文件引用为一层深度
* [ ] 适当使用了渐进式披露
* [ ] 工作流有清晰的步骤

### 代码和脚本

* [ ] 脚本解决问题而非推给 agent
* [ ] 错误处理明确且有帮助
* [ ] 没有"巫术常量"（所有值都有理由）
* [ ] 所需包列在指令中并验证为可用
* [ ] 脚本有清晰的文档
* [ ] 没有 Windows 风格路径（全部使用正斜杠）
* [ ] 关键操作有验证/确认步骤
* [ ] 质量关键任务包含反馈循环

### 测试

* [ ] 至少创建了三个评估
* [ ] 用 Haiku、Sonnet 和 Opus 测试过
* [ ] 用真实使用场景测试过
* [ ] 已纳入团队反馈（如适用）

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
