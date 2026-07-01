# 技能编写最佳实践

> 学习如何编写代理可以发现并成功使用的有效 Skills。

好的 Skills 要简洁、结构良好、经过实际使用测试。本指南提供实用的编写决策，帮助你编写代理可以发现并有效使用的 Skills。

关于 Skills 工作原理的概念背景，请参见 [Skills overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)。

## 核心原则

### 简洁是关键

[上下文窗口](https://platform.claude.com/docs/en/build-with-claude/context-windows)是公共资源。你的 Skill 与代理需要知道的所有其他内容共享上下文窗口，包括：

* 系统提示词
* 对话历史
* 其他 Skills 的元数据
* 你的实际请求

不是你的 Skill 中的每个 token 都有即时成本。在启动时，只有所有 Skills 的元数据（name 和 description）被预加载。代理仅在 Skill 变得相关时才读取 SKILL.md，并且仅在需要时读取额外的文件。然而，在 SKILL.md 中保持简洁仍然重要：一旦代理加载了它，每个 token 都与对话历史和其他上下文竞争。

**默认假设**：代理已经非常聪明

只添加代理没有的上下文。挑战每条信息：

* "代理真的需要这个解释吗？"
* "我可以假设代理知道这个吗？"
* "这个段落值得它的 token 成本吗？"

**好的示例：简洁**（约 50 个 tokens）：

````markdown  theme={null}
## 提取 PDF 文本

使用 pdfplumber 进行文本提取：

```python
import pdfplumber

with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```
````

**坏的示例：过于冗长**（约 150 个 tokens）：

```markdown  theme={null}
## 提取 PDF 文本

PDF（Portable Document Format）文件是一种常见的文件格式，包含
文本、图像和其他内容。要从 PDF 中提取文本，你需要
使用一个库。有很多可用的 PDF 处理库，但我们
推荐 pdfplumber，因为它易于使用且处理大多数情况很好。
首先，你需要使用 pip 安装它。然后你可以使用下面的代码...
```

简洁版本假设代理知道 PDF 是什么以及库如何工作。

### 设置适当的自由度

将具体程度与任务的脆弱性和可变性匹配。

**高自由度**（基于文本的指令）：

使用时机：

* 多种方法都是有效的
* 决策取决于上下文
* 启发式方法引导方法

示例：

```markdown  theme={null}
## 代码审查过程

1. 分析代码结构和组织
2. 检查潜在的 bug 或边缘情况
3. 提出可读性和可维护性的改进建议
4. 验证是否遵守项目约定
```

**中等自由度**（伪代码或带参数的脚本）：

使用时机：

* 存在优选模式
* 允许一些变化
* 配置影响行为

示例：

````markdown  theme={null}
## 生成报告

使用此模板并根据需要自定义：

```python
def generate_report(data, format="markdown", include_charts=True):
    # 处理数据
    # 以指定格式生成输出
    # 可选地包含可视化
```
````

**低自由度**（特定脚本，少量或没有参数）：

使用时机：

* 操作脆弱且容易出错
* 一致性至关重要
* 必须遵循特定顺序

示例：

````markdown  theme={null}
## 数据库迁移

准确运行此脚本：

```bash
python scripts/migrate.py --verify --backup
```

不要修改命令或添加额外标志。
````

**类比**：将代理想象成一个探索路径的机器人：

* **两边都是悬崖的窄桥**：只有一种安全的前进方式。提供具体的护栏和精确的指令（低自由度）。示例：必须按确切顺序运行的数据库迁移。
* **没有危险的开阔地带**：许多路径通向成功。给出一般方向，信任代理找到最佳路线（高自由度）。示例：上下文决定最佳方法的代码审查。

### 使用你计划使用的所有模型进行测试

Skills 作为模型的补充，因此有效性取决于底层模型。使用你计划使用的所有模型测试你的 Skill。

**按模型的测试考虑**：

* **Claude Haiku**（快速、经济）：Skill 是否提供足够的指导？
* **Claude Sonnet**（平衡）：Skill 是否清晰高效？
* **Claude Opus**（强大的推理能力）：Skill 是否避免了过度解释？

对 Opus 完美工作的内容可能需要对 Haiku 有更多细节。如果你计划跨多个模型使用 Skill，目标是使指令对所有模型都有效。

## Skill 结构

<Note>
  **YAML Frontmatter**：SKILL.md frontmatter 需要两个字段：

  * `name` - Skill 的人类可读名称（最多 64 个字符）
  * `description` - Skill 做什么和何时使用的单行描述（最多 1024 个字符）

  关于完整的 Skill 结构详细信息，请参见 [Skills overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#skill-structure)。
</Note>

### 命名约定

使用一致的命名模式使 Skills 更容易引用和讨论。我们建议对 Skill 名称使用**动名词形式**（动词 + -ing），因为这清楚地描述了 Skill 提供的活动或能力。

**好的命名示例（动名词形式）**：

* "Processing PDFs"
* "Analyzing spreadsheets"
* "Managing databases"
* "Testing code"
* "Writing documentation"

**可接受的替代方案**：

* 名词短语："PDF Processing"、"Spreadsheet Analysis"
* 行动导向："Process PDFs"、"Analyze Spreadsheets"

**避免**：

* 模糊的名称："Helper"、"Utils"、"Tools"
* 过于通用："Documents"、"Data"、"Files"
* 技能集合中不一致的模式

一致的命名使以下事情更容易：

* 在文档和对话中引用 Skills
* 一眼理解 Skill 做什么
* 组织和搜索多个 Skills
* 维护专业、一致的技能库

### 编写有效的描述

`description` 字段启用 Skill 发现，应包括 Skill 做什么和何时使用它。

<Warning>
  **始终用第三人称写**。描述被注入到系统提示词中，不一致的视角会导致发现问题。

  * **正确：** "Processes Excel files and generates reports"
  * **避免：** "I can help you process Excel files"
  * **避免：** "You can use this to process Excel files"
</Warning>

**具体且包含关键术语**。包括 Skill 做什么以及何时使用它的具体触发器/上下文。

每个 Skill 只有一个 description 字段。描述对技能选择至关重要：代理使用它在潜在的 100+ 个可用 Skills 中选择正确的 Skill。你的描述必须提供足够的细节让代理知道何时选择此 Skill，而 SKILL.md 的其余部分提供实现细节。

有效示例：

**PDF Processing skill：**

```yaml  theme={null}
description: Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
```

**Excel Analysis skill：**

```yaml  theme={null}
description: Analyze Excel spreadsheets, create pivot tables, generate charts. Use when analyzing Excel files, spreadsheets, tabular data, or .xlsx files.
```

**Git Commit Helper skill：**

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

SKILL.md 作为概述，根据需要将代理指向详细材料，就像入门指南中的目录。关于渐进式披露如何工作的解释，请参见概述中的 [How Skills work](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#how-skills-work)。

**实用指导：**

* 保持 SKILL.md 正文在 500 行以下以获得最佳性能
* 接近此限制时将内容拆分到单独的文件
* 使用下面的模式有效地组织指令、代码和资源

#### 可视化概述：从简单到复杂

一个基本 Skill 从仅包含元数据和指令的 SKILL.md 文件开始：

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=87782ff239b297d9a9e8e1b72ed72db9" alt="Simple SKILL.md file showing YAML frontmatter and markdown body" data-og-width="2048" width="2048" data-og-height="1153" height="1153" data-path="images/agent-skills-simple-file.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=c61cc33b6f5855809907f7fda94cd80e 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=90d2c0c1c76b36e8d485f49e0810dbfd 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=ad17d231ac7b0bea7e5b4d58fb4aeabb 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=f5d0a7a3c668435bb0aee9a3a8f8c329 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=0e927c1af9de5799cfe557d12249f6e6 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=46bbb1a51dd4c8202a470ac8c80a893d 2500w" />

随着你的 Skill 增长，你可以捆绑仅在需要时代理才加载的额外内容：

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=a5e0aa41e3d53985a7e3e43668a33ea3" alt="Bundling additional reference files like reference.md and forms.md." data-og-width="2048" width="2048" data-og-height="1327" height="1327" data-path="images/agent-skills-bundling-content.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=f8a0e73783e99b4a643d79eac86b70a2 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=dc510a2a9d3f14359416b706f067904a 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=82cd6286c966303f7dd914c28170e385 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=56f3be36c77e4fe4b523df209a6824c6 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=d22b5161b2075656417d56f41a74f3dd 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=3dd4bdd6850ffcc96c6c45fcb0acd6eb 2500w" />

完整的 Skill 目录结构可能如下：

```
pdf/
├── SKILL.md              # 主指令（触发时加载）
├── FORMS.md              # 表单填充指南（按需加载）
├── reference.md          # API 参考（按需加载）
├── examples.md           # 使用示例（按需加载）
└── scripts/
    ├── analyze_form.py   # 工具脚本（执行，不加载到上下文）
    ├── fill_form.py      # 表单填充脚本
    └── validate.py       # 验证脚本
```

#### 模式 1：带参考的高层指南

````markdown  theme={null}
---
name: PDF Processing
description: Extracts text and tables from PDF files, fills forms, and merges documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
---

# PDF Processing

## 快速开始

使用 pdfplumber 提取文本：
```python
import pdfplumber
with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```

## 高级功能

**表单填充**：完整指南见 [FORMS.md](FORMS.md)
**API 参考**：所有方法见 [REFERENCE.md](REFERENCE.md)
**示例**：常见模式见 [EXAMPLES.md](EXAMPLES.md)
````

代理仅在需要时加载 FORMS.md、REFERENCE.md 或 EXAMPLES.md。

#### 模式 2：按领域组织

对于有多个领域的 Skills，按领域组织内容以避免加载无关上下文。当用户询问销售指标时，代理只需要读取销售相关的模式，而不是财务或营销数据。这保持 token 使用低且上下文专注。

```
bigquery-skill/
├── SKILL.md（概述和导航）
└── reference/
    ├── finance.md（收入、计费指标）
    ├── sales.md（商机、管道）
    ├── product.md（API 使用、功能）
    └── marketing.md（活动、归因）
```

````markdown SKILL.md theme={null}
# BigQuery 数据分析

## 可用数据集

**财务**：收入、ARR、计费 → 见 [reference/finance.md](reference/finance.md)
**销售**：商机、管道、账户 → 见 [reference/sales.md](reference/sales.md)
**产品**：API 使用、功能、采用 → 见 [reference/product.md](reference/product.md)
**营销**：活动、归因、邮件 → 见 [reference/marketing.md](reference/marketing.md)

## 快速搜索

使用 grep 查找特定指标：

```bash
grep -i "revenue" reference/finance.md
grep -i "pipeline" reference/sales.md
grep -i "api usage" reference/product.md
```
````

#### 模式 3：条件性细节

显示基本内容，链接到高级内容：

```markdown  theme={null}
# DOCX 处理

## 创建文档

使用 docx-js 创建新文档。见 [DOCX-JS.md](DOCX-JS.md)。

## 编辑文档

对于简单编辑，直接修改 XML。

**对于修订追踪**：见 [REDLINING.md](REDLINING.md)
**对于 OOXML 细节**：见 [OOXML.md](OOXML.md)
```

代理仅在用户需要这些功能时读取 REDLINING.md 或 OOXML.md。

### 避免深度嵌套的引用

当代理从其他引用的文件中再次引用文件时，可能会部分读取文件。遇到嵌套引用时，代理可能使用 `head -100` 等命令预览内容而不是读取整个文件，导致信息不完整。

**保持引用从 SKILL.md 起一层深度**。所有引用文件应直接从 SKILL.md 链接，确保代理在需要时读取完整文件。

**坏的示例：太深**：

```markdown  theme={null}
# SKILL.md
见 [advanced.md](advanced.md)...

# advanced.md
见 [details.md](details.md)...

# details.md
这里才是实际信息...
```

**好的示例：仅一层深度**：

```markdown  theme={null}
# SKILL.md

**基本用法**：[SKILL.md 中的指令]
**高级功能**：见 [advanced.md](advanced.md)
**API 参考**：见 [reference.md](reference.md)
**示例**：见 [examples.md](examples.md)
```

### 为较长的引用文件添加目录

对于超过 100 行的引用文件，在顶部包含一个目录。这确保代理即使通过部分读取预览也能看到可用信息的全貌。

**示例**：

```markdown  theme={null}
# API 参考

## 目录
- 认证和设置
- 核心方法（创建、读取、更新、删除）
- 高级功能（批量操作、webhooks）
- 错误处理模式
- 代码示例

## 认证和设置
...

## 核心方法
...
```

代理可以根据需要读取完整文件或跳转到特定部分。

关于这种基于文件系统的架构如何启用渐进式披露的详细信息，请参见下方高级部分中的 [运行时环境](#runtime-environment)。

## 工作流和反馈循环

### 为复杂任务使用工作流

将复杂操作分解为清晰的顺序步骤。对于特别复杂的工作流，提供代理可以复制到其响应中并在进展时勾选的检查清单。

**示例 1：研究合成工作流**（用于没有代码的 Skills）：

````markdown  theme={null}
## 研究合成工作流

复制此检查清单并跟踪你的进展：

```
研究进展：
- [ ] 第 1 步：阅读所有源文档
- [ ] 第 2 步：识别关键主题
- [ ] 第 3 步：交叉引用声明
- [ ] 第 4 步：创建结构化摘要
- [ ] 第 5 步：验证引用
```

**第 1 步：阅读所有源文档**

查看 `sources/` 目录中的每个文档。记录主要论点和支持证据。

**第 2 步：识别关键主题**

查找跨源的模式。哪些主题反复出现？源之间在哪里一致或不同？

**第 3 步：交叉引用声明**

对于每个主要声明，验证它是否出现在源材料中。记录哪个源支持每一点。

**第 4 步：创建结构化摘要**

按主题组织发现。包括：
- 主要声明
- 来自源的支持证据
- 冲突的观点（如有）

**第 5 步：验证引用**

检查每个声明都引用了正确的源文档。如果引用不完整，返回第 3 步。
````

此示例显示了工作流如何应用于不需要代码的分析任务。检查清单模式适用于任何复杂的多步骤过程。

**示例 2：PDF 表单填充工作流**（用于有代码的 Skills）：

````markdown  theme={null}
## PDF 表单填充工作流

复制此检查清单并在完成时勾选：

```
任务进展：
- [ ] 第 1 步：分析表单（运行 analyze_form.py）
- [ ] 第 2 步：创建字段映射（编辑 fields.json）
- [ ] 第 3 步：验证映射（运行 validate_fields.py）
- [ ] 第 4 步：填充表单（运行 fill_form.py）
- [ ] 第 5 步：验证输出（运行 verify_output.py）
```

**第 1 步：分析表单**

运行：`python scripts/analyze_form.py input.pdf`

这提取表单字段及其位置，保存到 `fields.json`。

**第 2 步：创建字段映射**

编辑 `fields.json` 为每个字段添加值。

**第 3 步：验证映射**

运行：`python scripts/validate_fields.py fields.json`

在继续前修复任何验证错误。

**第 4 步：填充表单**

运行：`python scripts/fill_form.py input.pdf fields.json output.pdf`

**第 5 步：验证输出**

运行：`python scripts/verify_output.py output.pdf`

如果验证失败，返回第 2 步。
````

清晰的步骤防止代理跳过关键验证。检查清单帮助你和代理跟踪多步骤工作流的进展。

### 实现反馈循环

**常见模式**：运行验证器 → 修复错误 → 重复

此模式大大提高了输出质量。

**示例 1：风格指南合规**（用于没有代码的 Skills）：

```markdown  theme={null}
## 内容审查过程

1. 按照 STYLE_GUIDE.md 中的指南起草内容
2. 对照检查清单审查：
   - 检查术语一致性
   - 验证示例遵循标准格式
   - 确认所有必需的部分都存在
3. 如果发现问题：
   - 用具体的部分引用记录每个问题
   - 修订内容
   - 再次审查检查清单
4. 仅在满足所有要求后才继续
5. 定稿并保存文档
```

这展示了使用参考文档而非脚本的验证循环模式。"验证器"是 STYLE_GUIDE.md，代理通过阅读和比较来执行检查。

**示例 2：文档编辑过程**（用于有代码的 Skills）：

```markdown  theme={null}
## 文档编辑过程

1. 对 `word/document.xml` 进行编辑
2. **立即验证**：`python ooxml/scripts/validate.py unpacked_dir/`
3. 如果验证失败：
   - 仔细查看错误信息
   - 修复 XML 中的问题
   - 再次运行验证
4. **仅在验证通过后才继续**
5. 重新构建：`python ooxml/scripts/pack.py unpacked_dir/ output.docx`
6. 测试输出文档
```

验证循环及早捕获错误。

## 内容指南

### 避免时间敏感信息

不要包含会过时的信息：

**坏的示例：时间敏感**（会变得错误）：

```markdown  theme={null}
如果你在 2025 年 8 月之前做这件事，使用旧 API。
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

旧模式部分提供历史上下文而不使主要内容混乱。

### 使用一致的术语

选择一个术语并在整个 Skill 中使用：

**好的 - 一致**：

* 始终 "API endpoint"
* 始终 "field"
* 始终 "extract"

**坏的 - 不一致**：

* 混用 "API endpoint"、"URL"、"API route"、"path"
* 混用 "field"、"box"、"element"、"control"
* 混用 "extract"、"pull"、"get"、"retrieve"

一致性帮助代理理解和遵循指令。

## 常见模式

### 模板模式

为输出格式提供模板。将严格程度与你的需求匹配。

**对于严格要求**（如 API 响应或数据格式）：

````markdown  theme={null}
## 报告结构

始终使用此确切模板结构：

```markdown
# [分析标题]

## 执行摘要
[关键发现的单段概述]

## 关键发现
- 发现 1 及支持数据
- 发现 2 及支持数据
- 发现 3 及支持数据

## 建议
1. 具体可操作的建议
2. 具体可操作的建议
```
````

**对于灵活指导**（当改编有用时）：

````markdown  theme={null}
## 报告结构

这是一个合理的默认格式，但根据分析使用你的最佳判断：

```markdown
# [分析标题]

## 执行摘要
[概述]

## 关键发现
[根据你发现的内容调整部分]

## 建议
[根据具体上下文定制]
```

根据特定分析类型调整部分。
````

### 示例模式

对于输出质量取决于看到示例的 Skills，提供输入/输出对，就像在常规提示词中一样：

````markdown  theme={null}
## 提交消息格式

按照这些示例生成提交消息：

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

遵循此风格：type(scope): 简要描述，然后是详细解释。
````

示例帮助代理比仅靠描述更清晰地理解所需的风格和细节水平。

### 条件工作流模式

引导代理通过决策点：

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
  如果工作流变得庞大或复杂，包含许多步骤，考虑将它们推送到单独的文件中，并告诉代理根据手头任务读取适当的文件。
</Tip>

## 评估和迭代

### 先构建评估

**在编写大量文档之前创建评估。** 这确保你的 Skill 解决真实问题而不是记录想象中的问题。

**评估驱动开发：**

1. **识别差距**：在没有 Skill 的情况下在代表性任务上运行你的代理。记录具体失败或缺失的上下文
2. **创建评估**：构建三个测试这些差距的场景
3. **建立基线**：测量代理在没有 Skill 的情况下的表现
4. **编写最小指令**：创建刚好足够的内容来解决差距并通过评估
5. **迭代**：执行评估，与基线比较，并优化

这种方法确保你解决实际问题，而不是预期可能永远不会出现的要求。

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
  此示例演示了带有简单测试评分标准的数据驱动评估。我们目前不提供运行这些评估的内置方式。用户可以创建自己的评估系统。评估是你衡量 Skill 有效性的真相来源。
</Note>

### 与代理迭代地开发 Skills

最有效的 Skill 开发过程涉及代理本身。与一个实例（"Agent A"）一起工作创建将由其他实例（"Agent B"）使用的 Skill。Agent A 帮助你设计和优化指令，而 Agent B 在实际任务中测试它们。这有效是因为底层模型既理解如何编写有效的代理指令，也理解代理需要什么信息。

**创建新 Skill：**

1. **在没有 Skill 的情况下完成任务**：使用正常提示词与 Agent A 解决一个问题。在工作过程中，你自然地提供上下文、解释偏好并分享过程性知识。注意你反复提供了什么信息。

2. **识别可重用模式**：完成任务后，识别你提供的哪些上下文对类似未来任务有用。

   **示例**：如果你完成了一个 BigQuery 分析，你可能提供了表名、字段定义、过滤规则（如"始终排除测试账户"）和常见查询模式。

3. **请 Agent A 创建 Skill**："创建一个捕获我们刚使用的 BigQuery 分析模式的 Skill。包括表模式、命名约定和过滤测试账户的规则。"

   <Tip>
     现代代理原生理解 Skill 格式和结构。你不需要特殊的系统提示词或"编写技能"技能来获得创建 Skills 的帮助。只需请代理创建一个 Skill，它将生成正确结构的 SKILL.md 内容，包含适当的 frontmatter 和正文内容。
   </Tip>

4. **审查简洁性**：检查 Agent A 没有添加不必要的解释。询问："删除关于 win rate 含义的解释——代理已经知道那是什么。"

5. **改进信息架构**：请 Agent A 更有效地组织内容。例如："组织这些使表模式在单独的引用文件中。我们以后可能添加更多表。"

6. **在类似任务上测试**：在相关用例上与 Agent B（加载了 Skill 的全新实例）一起使用 Skill。观察 Agent B 是否找到正确的信息、正确应用规则并成功处理任务。

7. **基于观察迭代**：如果 Agent B 遇到困难或遗漏了什么，带着具体细节返回 Agent A："当代理使用此 Skill 时，它忘记了按 Q4 的日期过滤。我们应该添加关于日期过滤模式的部分吗？"

**迭代现有 Skills：**

同样的层级模式在改进 Skills 时继续。你在以下之间交替：

* **与 Agent A 一起工作**（帮助优化 Skill 的专家）
* **与 Agent B 一起测试**（使用 Skill 执行实际工作的代理）
* **观察 Agent B 的行为**并将洞察带回 Agent A

1. **在实际工作流中使用 Skill**：给 Agent B（加载了 Skill）实际任务，不是测试场景

2. **观察 Agent B 的行为**：注意它在哪里挣扎、成功或做出意外选择

   **示例观察**："当我请 Agent B 做区域销售报告时，它写了查询但忘记了过滤测试账户，即使 Skill 提到了这个规则。"

3. **返回 Agent A 进行改进**：分享当前的 SKILL.md 并描述你观察到的内容。问："我注意到当我请求区域报告时 Agent B 忘记了过滤测试账户。Skill 提到了过滤，但也许不够突出？"

4. **审查 Agent A 的建议**：Agent A 可能建议重新组织以使规则更突出，使用更强的语言如"必须过滤"而不是"始终过滤"，或重构工作流部分。

5. **应用并测试更改**：用 Agent A 的优化更新 Skill，然后在类似请求上与 Agent B 再次测试

6. **基于使用情况重复**：在遇到新场景时继续这个观察-优化-测试循环。每次迭代基于真实代理行为改进 Skill，而不是假设。

**收集团队反馈：**

1. 与队友分享 Skills 并观察他们的使用
2. 询问：Skill 在预期时激活了吗？指令清晰吗？缺少什么？
3. 整合反馈以解决你自己使用模式中的盲点

**为什么这种方法有效**：Agent A 理解代理需求，你提供领域专业知识，Agent B 通过真实使用揭示差距，迭代优化基于观察到的行为而不是假设来改进 Skills。

### 观察代理如何导航 Skills

在迭代 Skills 时，注意代理在实践中实际如何使用它们。留意：

* **意外的探索路径**：代理是否以你未预期的顺序读取文件？这可能表明你的结构不如你认为的直观
* **遗漏的连接**：代理是否未能遵循对重要文件的引用？你的链接可能需要更明确或突出
* **对某些部分的过度依赖**：如果代理反复读取相同的文件，考虑该内容是否应该放在主 SKILL.md 中
* **被忽略的内容**：如果代理从未访问绑定的文件，它可能是不必要的或在主指令中没有得到充分

基于这些观察而不是假设进行迭代。Skill 元数据中的 'name' 和 'description' 尤其关键。代理在决定是否触发 Skill 以响应当前任务时使用它们。确保它们清楚地描述 Skill 做什么以及何时应该使用。

## 应避免的反模式

### 避免 Windows 风格的路径

即使在 Windows 上，也始终在文件路径中使用正斜杠：

* ✓ **正确**：`scripts/helper.py`、`reference/guide.md`
* ✗ **避免**：`scripts\helper.py`、`reference\guide.md`

Unix 风格路径跨所有平台工作，而 Windows 风格路径在 Unix 系统上会导致错误。

### 避免提供太多选项

除非必要，不要呈现多种方法：

````markdown  theme={null}
**坏的示例：太多选择**（令人困惑）：
"你可以使用 pypdf，或 pdfplumber，或 PyMuPDF，或 pdf2image，或..."

**好的示例：提供默认值**（带逃生出口）：
"使用 pdfplumber 进行文本提取：
```python
import pdfplumber
```

对于需要 OCR 的扫描 PDF，改用 pdf2image 配合 pytesseract。"
````

## 高级：带可执行代码的 Skills

以下部分专注于包含可执行脚本的 Skills。如果你的 Skill 仅使用 markdown 指令，跳到 [有效 Skills 的检查清单](#checklist-for-effective-skills)。

### 解决问题，不要推卸

在为 Skills 编写脚本时，处理错误条件而不是将其推卸给代理。

**好的示例：显式处理错误**：

```python  theme={null}
def process_file(path):
    """处理文件，如果不存在则创建。"""
    try:
        with open(path) as f:
            return f.read()
    except FileNotFoundError:
        # 创建带默认内容的文件而不是失败
        print(f"文件 {path} 未找到，创建默认内容")
        with open(path, 'w') as f:
            f.write('')
        return ''
    except PermissionError:
        # 提供替代方案而不是失败
        print(f"无法访问 {path}，使用默认值")
        return ''
```

**坏的示例：推卸给代理**：

```python  theme={null}
def process_file(path):
    # 只是失败并让代理自己处理
    return open(path).read()
```

配置参数也应该有理由和文档记录，以避免"巫术常量"（Ousterhout 定律）。如果你不知道正确的值，代理将如何确定它？

**好的示例：自我文档化**：

```python  theme={null}
# HTTP 请求通常在 30 秒内完成
# 更长的超时时间考虑到慢速连接
REQUEST_TIMEOUT = 30

# 三次重试平衡可靠性与速度
# 大多数间歇性故障在第二次重试时解决
MAX_RETRIES = 3
```

**坏的示例：魔法数字**：

```python  theme={null}
TIMEOUT = 47  # 为什么是 47？
RETRIES = 5   # 为什么是 5？
```

### 提供实用脚本

即使你的代理可以编写脚本，预制的脚本也提供优势：

**实用脚本的好处**：

* 比生成的代码更可靠
* 节省 tokens（不需要将代码包含在上下文中）
* 节省时间（不需要生成代码）
* 确保跨使用的一致性

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=4bbc45f2c2e0bee9f2f0d5da669bad00" alt="Bundling executable scripts alongside instruction files" data-og-width="2048" width="2048" data-og-height="1154" height="1154" data-path="images/agent-skills-executable-scripts.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=9a04e6535a8467bfeea492e517de389f 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=e49333ad90141af17c0d7651cca7216b 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=954265a5df52223d6572b6214168c428 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=2ff7a2d8f2a83ee8af132b29f10150fd 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=48ab96245e04077f4d15e9170e081cfb 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=0301a6c8b3ee879497cc5b5483177c90 2500w" />

上图显示了可执行脚本如何与指令文件一起工作。指令文件（forms.md）引用脚本，代理可以执行它而不将其内容加载到上下文中。

**重要区别**：在你的指令中明确说明代理是否应该：

* **执行脚本**（最常见）："运行 `analyze_form.py` 提取字段"
* **将其作为参考阅读**（用于复杂逻辑）："参见 `analyze_form.py` 了解字段提取算法"

对于大多数实用脚本，执行是首选的，因为它更可靠和高效。关于脚本执行如何工作的详细信息，请参见下方的 [运行时环境](#runtime-environment) 部分。

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

**validate_boxes.py**：检查重叠的边界框

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

当输入可以渲染为图像时，让代理分析它们：

````markdown  theme={null}
## 表单布局分析

1. 将 PDF 转换为图像：
   ```bash
   python scripts/pdf_to_images.py form.pdf
   ```

2. 分析每个页面图像以识别表单字段
3. 代理可以通过视觉看到字段位置和类型
````

<Note>
  在此示例中，你需要编写 `pdf_to_images.py` 脚本。
</Note>

代理视觉能力有助于理解布局和结构。

### 创建可验证的中间输出

当代理执行复杂的开放性任务时，它们可能犯错。"计划-验证-执行"模式通过在代理首先以结构化格式创建计划，然后在执行前使用脚本验证该计划，来及早捕获错误。

**示例**：想象要求代理根据电子表格更新 PDF 中的 50 个表单字段。没有验证，它可能引用不存在的字段、创建冲突的值、遗漏必需字段或不正确地应用更新。

**解决方案**：使用上面显示的工作流模式（PDF 表单填充），但添加一个在执行更改前被验证的中间 `changes.json` 文件。工作流变为：分析 → **创建计划文件** → **验证计划** → 执行 → 验证。

**为什么此模式有效：**

* **及早捕获错误**：验证在应用更改前发现问题
* **机器可验证**：脚本提供客观验证
* **可逆计划**：代理可以在不触碰原件的情况下迭代计划
* **清晰的调试**：错误消息指向具体问题

**何时使用**：批量操作、破坏性更改、复杂验证规则、高风险操作。

**实现提示**：使验证脚本详细输出具体错误消息，如 "字段 'signature_date' 未找到。可用字段：customer_name、order_total、signature_date_signed" 以帮助代理修复问题。

### 打包依赖

Skills 在代码执行环境中运行，具有平台特定的限制：

* **claude.ai**：可以从 npm 和 PyPI 安装包，并可以从 GitHub 仓库拉取
* **Anthropic API**：没有网络访问权限，没有运行时包安装

在你的 SKILL.md 中列出必需的包，并验证它们在 [代码执行工具文档](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool) 中是否可用。

### 运行时环境

Skills 在具有文件系统访问权限、bash 命令和代码执行能力的代码执行环境中运行。关于此架构的概念解释，请参见概述中的 [The Skills architecture](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#the-skills-architecture)。

**这对你的编写有何影响：**

**代理如何访问 Skills：**

1. **元数据预加载**：启动时，所有 Skills 的 YAML frontmatter 中的 name 和 description 被加载到系统提示词中
2. **文件按需读取**：代理在需要时使用文件读取工具从文件系统访问 SKILL.md 和其他文件
3. **脚本高效执行**：实用脚本可以通过 bash 执行，而不将完整内容加载到上下文中。只有脚本的输出消耗 tokens
4. **大文件无上下文损失**：引用文件、数据或文档在实际被读取之前不消耗上下文 tokens

* **文件路径很重要**：代理像文件系统一样导航你的 skill 目录。使用正斜杠（`reference/guide.md`），而不是反斜杠
* **描述性命文件名**：使用指示内容的名称：`form_validation_rules.md`，不是 `doc2.md`
* **为发现而组织**：按领域或功能组织目录
  * 好：`reference/finance.md`、`reference/sales.md`
  * 坏：`docs/file1.md`、`docs/file2.md`
* **捆绑全面的资源**：包含完整的 API 文档、大量示例、大数据集；在被访问之前没有上下文损失
* **优先使用脚本进行确定性操作**：编写 `validate_form.py` 而不是要求代理生成验证代码
* **使执行意图清晰**：
  * "运行 `analyze_form.py` 提取字段"（执行）
  * "参见 `analyze_form.py` 了解提取算法"（作为参考阅读）
* **测试文件访问模式**：通过真实请求测试验证代理是否可以导航你的目录结构

**示例：**

```
bigquery-skill/
├── SKILL.md（概述，指向引用文件）
└── reference/
    ├── finance.md（收入指标）
    ├── sales.md（管道数据）
    └── product.md（使用分析）
```

当用户询问收入时，代理读取 SKILL.md，看到对 `reference/finance.md` 的引用，并调用 bash 仅读取该文件。sales.md 和 product.md 文件保留在文件系统上，在需要之前消耗零个上下文 tokens。这种基于文件系统的模型是启用渐进式披露的原因。代理可以导航并有选择地加载每个任务所需的内容。

关于技术架构的完整详细信息，请参见 Skills 概述中的 [How Skills work](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#how-skills-work)。

### MCP 工具引用

如果你的 Skill 使用 MCP（Model Context Protocol）工具，始终使用完全限定的工具名称以避免"未找到工具"错误。

**格式**：`ServerName:tool_name`

**示例**：

```markdown  theme={null}
使用 BigQuery:bigquery_schema 工具获取表模式。
使用 GitHub:create_issue 工具创建 issue。
```

其中：

* `BigQuery` 和 `GitHub` 是 MCP 服务器名称
* `bigquery_schema` 和 `create_issue` 是这些服务器中的工具名称

没有服务器前缀，代理可能无法定位工具，特别是在有多个 MCP 服务器可用时。

### 避免假设工具已安装

不要假设包已可用：

````markdown  theme={null}
**坏的示例：假设已安装**：
"使用 pdf 库处理文件。"

**好的示例：明确说明依赖**：
"安装必需的包：`pip install pypdf`

然后使用它：
```python
from pypdf import PdfReader
reader = PdfReader("file.pdf")
```"
````

## 技术说明

### YAML frontmatter 要求

SKILL.md frontmatter 需要 `name`（最多 64 个字符）和 `description`（最多 1024 个字符）字段。完整结构详细信息请参见 [Skills overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#skill-structure)。

### Token 预算

保持 SKILL.md 正文在 500 行以下以获得最佳性能。如果你内容超过此限制，使用前面描述的渐进式披露模式将其拆分到单独的文件中。关于架构详细信息，请参见 [Skills overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#how-skills-work)。

## 有效 Skills 的检查清单

在分享 Skill 之前，验证：

### 核心质量

* [ ] 描述具体且包含关键术语
* [ ] 描述包括 Skill 做什么和何时使用它
* [ ] SKILL.md 正文在 500 行以下
* [ ] 额外细节在单独的文件中（如需要）
* [ ] 没有时间敏感信息（或在"旧模式"部分）
* [ ] 全文中术语一致
* [ ] 示例是具体的，不是抽象的
* [ ] 文件引用一层深度
* [ ] 适当使用渐进式披露
* [ ] 工作流有清晰的步骤

### 代码和脚本

* [ ] 脚本解决问题而不是推卸给代理
* [ ] 错误处理是显式且有帮助的
* [ ] 没有"巫术常量"（所有值都有理由）
* [ ] 所需包在指令中列出并验证为可用
* [ ] 脚本有清晰的文档
* [ ] 没有 Windows 风格的路径（全部正斜杠）
* [ ] 关键操作有验证/验证步骤
* [ ] 质量关键任务包含反馈循环

### 测试

* [ ] 至少创建了三个评估
* [ ] 使用 Haiku、Sonnet 和 Opus 进行了测试
* [ ] 使用真实使用场景进行了测试
* [ ] 团队反馈已整合（如适用）

## 下一步

<CardGroup cols={2}>
  <Card title="Get started with Agent Skills" icon="rocket" href="https://platform.claude.com/docs/en/agents-and-tools/agent-skills/quickstart">
    创建你的第一个 Skill
  </Card>

  <Card title="Use Skills in Claude Code" icon="terminal" href="https://code.claude.com/docs/en/skills">
    在 Claude Code 中创建和管理 Skills
  </Card>

  <Card title="Use Skills with the API" icon="code" href="https://platform.claude.com/docs/en/build-with-claude/skills-guide">
    以编程方式上传和使用 Skills
  </Card>
</CardGroup>
