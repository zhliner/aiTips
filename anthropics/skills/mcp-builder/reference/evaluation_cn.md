# MCP Server 评估指南

## 概述

本文档提供创建 MCP server 综合评估的指导。评估用于测试 LLM 是否能有效使用你的 MCP server，仅通过提供的 tool 来回答真实的、复杂的问题。

---

## 快速参考

### 评估要求
- 创建 10 道人类可读的问题
- 问题必须是只读、独立、非破坏性的
- 每道问题需要多次 tool 调用（可能需要数十次）
- 答案必须是单一的、可验证的值
- 答案必须是稳定的（不会随时间变化）

### 输出格式
```xml
<evaluation>
   <qa_pair>
      <question>Your question here</question>
      <answer>Single verifiable answer</answer>
   </qa_pair>
</evaluation>
```

---

## 评估的目的

衡量 MCP server 质量的标准不是 server 实现 tool 的完善程度或全面程度，而是这些实现（输入/输出 schema、docstring/描述、功能）在多大程度上帮助 LLM 在没有任何其他上下文、仅能访问 MCP server 的情况下，回答真实且困难的问题。

## 评估概述

创建 10 道人类可读的问题，仅需要只读、独立、非破坏性和幂等的操作来回答。每道问题应当：
- 真实
- 清晰简洁
- 无歧义
- 复杂，可能需要数十次 tool 调用或步骤
- 可以用你预先确定的单一、可验证的值来回答

## 问题指南

### 核心要求

1. **问题必须是独立的**
   - 每道问题不应依赖于任何其他问题的答案
   - 不应假设处理其他问题时已执行了先前的写操作

2. **问题必须仅需要非破坏性和幂等的 tool 使用**
   - 不应要求或指示修改状态来得出正确答案

3. **问题必须是真实的、清晰的、简洁的和复杂的**
   - 必须要求另一个 LLM 使用多个（可能需要数十个）tool 或步骤来回答

### 复杂性与深度

4. **问题必须需要深入探索**
   - 考虑需要多个子问题和顺序 tool 调用的多跳问题
   - 每个步骤都应从之前问题中发现的信息中受益

5. **问题可能需要大量分页**
   - 可能需要翻阅多页结果
   - 可能需要查询旧数据（1-2 年前的）来查找小众信息
   - 问题必须是困难的

6. **问题必须需要深入理解**
   - 而非表面知识
   - 可以提出复杂的 True/False 问题并要求提供证据
   - 可以使用多选题格式，让 LLM 搜索不同的假设

7. **问题不能通过简单的关键词搜索解决**
   - 不要包含目标内容中的特定关键词
   - 使用同义词、相关概念或释义
   - 需要多次搜索、分析多个相关项目、提取上下文，然后推导答案

### Tool 测试

8. **问题应对 tool 返回值进行压力测试**
   - 可能导致 tool 返回大型 JSON 对象或列表，使 LLM 不堪重负
   - 应需要理解多种数据模态：
     - ID 和名称
     - 时间戳和日期时间（月、日、年、秒）
     - 文件 ID、名称、扩展名和 MIME 类型
     - URL、GID 等
   - 应探测 tool 返回所有有用数据形式的能力

9. **问题应主要反映真实的人类用例**
   - 即由 LLM 辅助的人类会关心的信息检索任务类型

10. **问题可能需要数十次 tool 调用**
    - 这对上下文有限的 LLM 构成挑战
    - 鼓励 MCP server tool 减少返回的信息量

11. **包含模糊性问题**
    - 可能是模糊的，或者在调用哪些 tool 上需要做困难的决策
    - 迫使 LLM 可能犯错或误解
    - 确保尽管存在模糊性，仍然有单一的可验证答案

### 稳定性

12. **问题的设计必须确保答案不会变化**
    - 不要依赖"当前状态"这类动态数据提问
    - 例如，不要统计：
      - 帖子的反应数量
      - 话题的回复数量
      - 频道的成员数量

13. **不要让 MCP server 限制你创建的问题类型**
    - 创建具有挑战性和复杂性的问题
    - 有些问题可能无法用现有的 MCP server tool 解决
    - 问题可能需要特定的输出格式（datetime vs. epoch time，JSON vs. MARKDOWN）
    - 问题可能需要数十次 tool 调用才能完成

## 答案指南

### 验证

1. **答案必须可通过直接字符串比较进行验证**
   - 如果答案可以用多种格式重写，请在问题中明确指定输出格式
   - 示例："使用 YYYY/MM/DD 格式。"、"回答 True 或 False。"、"只回答 A、B、C 或 D。"
   - 答案应为单一的可验证值，例如：
     - 用户 ID、用户名、显示名称、名字、姓氏
     - 频道 ID、频道名称
     - 消息 ID、字符串
     - URL、标题
     - 数值量
     - 时间戳、日期时间
     - 布尔值（用于 True/False 问题）
     - 电子邮件地址、电话号码
     - 文件 ID、文件名、文件扩展名
     - 多选题答案
   - 答案不应需要特殊格式或复杂的结构化输出
   - 答案将使用直接字符串比较进行验证

### 可读性

2. **答案通常应优先使用人类可读的格式**
   - 示例：名称、名字、姓氏、日期时间、文件名、消息字符串、URL、yes/no、true/false、a/b/c/d
   - 而非不透明的 ID（尽管 ID 也可以接受）
   - 绝大多数答案应该是人类可读的

### 稳定性

3. **答案必须是稳定/静态的**
   - 查看旧内容（例如已结束的对话、已发布的项目、已回答的问题）
   - 基于"已关闭"的概念创建问题，这些问题始终返回相同的答案
   - 问题可以要求考虑一个固定的时间窗口，以隔离非静态答案
   - 依赖不太可能变化的上下文
   - 示例：如果查找论文名称，要足够具体，以免答案与后来发表的论文混淆

4. **答案必须清晰且无歧义**
   - 问题的设计必须确保有单一的、明确的答案
   - 答案可以通过使用 MCP server tool 推导得出

### 多样性

5. **答案必须多样化**
   - 答案应为不同模态和格式的单一可验证值
   - 用户概念：用户 ID、用户名、显示名称、名字、姓氏、电子邮件地址、电话号码
   - 频道概念：频道 ID、频道名称、频道主题
   - 消息概念：消息 ID、消息字符串、时间戳、月、日、年

6. **答案不能是复杂结构**
   - 不是值列表
   - 不是复杂对象
   - 不是 ID 或字符串列表
   - 不是自然语言文本
   - 除非答案可以通过直接字符串比较直接验证
   - 且可以现实地复现
   - LLM 以其他顺序或格式返回相同列表的可能性应很低

## 评估流程

### 第 1 步：文档检查

阅读目标 API 的文档以了解：
- 可用的 endpoint 和功能
- 如果存在歧义，从 web 获取额外信息
- 尽可能并行化此步骤
- 确保每个子 agent 仅从文件系统或 web 查看文档

### 第 2 步：Tool 检查

列出 MCP server 中可用的 tool：
- 直接检查 MCP server
- 了解输入/输出 schema、docstring 和描述
- 在此阶段不要调用 tool 本身

### 第 3 步：建立理解

重复第 1 步和第 2 步，直到你有了充分的理解：
- 多次迭代
- 思考你想创建的任务类型
- 完善你的理解
- 在任何阶段都不要阅读 MCP server 实现本身的代码
- 使用你的直觉和理解来创建合理的、真实的但非常有挑战性的任务

### 第 4 步：只读内容检查

在了解 API 和 tool 之后，使用 MCP server tool：
- 仅使用只读和非破坏性操作检查内容
- 目标：识别特定内容（例如用户、频道、消息、项目、任务）以创建真实的问题
- 不应调用任何修改状态的 tool
- 不会阅读 MCP server 实现本身的代码
- 使用独立的子 agent 并行执行独立探索
- 确保每个子 agent 仅执行只读、非破坏性和幂等操作
- 注意：某些 tool 可能返回大量数据，导致上下文耗尽
- 进行增量、小范围、有针对性的 tool 调用来探索
- 在所有 tool 调用请求中，使用 `limit` 参数限制结果（<10）
- 使用分页

### 第 5 步：任务生成

在检查内容之后，创建 10 道人类可读的问题：
- LLM 应该能够通过 MCP server 回答这些问题
- 遵循上述所有问题和答案指南

## 输出格式

每个 QA 对由一个问题和一个答案组成。输出应为具有以下结构的 XML 文件：

```xml
<evaluation>
   <qa_pair>
      <question>Find the project created in Q2 2024 with the highest number of completed tasks. What is the project name?</question>
      <answer>Website Redesign</answer>
   </qa_pair>
   <qa_pair>
      <question>Search for issues labeled as "bug" that were closed in March 2024. Which user closed the most issues? Provide their username.</question>
      <answer>sarah_dev</answer>
   </qa_pair>
   <qa_pair>
      <question>Look for pull requests that modified files in the /api directory and were merged between January 1 and January 31, 2024. How many different contributors worked on these PRs?</question>
      <answer>7</answer>
   </qa_pair>
   <qa_pair>
      <question>Find the repository with the most stars that was created before 2023. What is the repository name?</question>
      <answer>data-pipeline</answer>
   </qa_pair>
</evaluation>
```

## 评估示例

### 好的问题

**示例 1：需要深入探索的多跳问题（GitHub MCP）**
```xml
<qa_pair>
   <question>Find the repository that was archived in Q3 2023 and had previously been the most forked project in the organization. What was the primary programming language used in that repository?</question>
   <answer>Python</answer>
</qa_pair>
```

这个问题好的原因：
- 需要多次搜索来查找已归档的仓库
- 需要识别归档前哪个项目的 fork 数最多
- 需要检查仓库详情以获取语言信息
- 答案是简单的、可验证的值
- 基于不会变化的历史（已关闭）数据

**示例 2：需要理解上下文而非关键词匹配（项目管理 MCP）**
```xml
<qa_pair>
   <question>Locate the initiative focused on improving customer onboarding that was completed in late 2023. The project lead created a retrospective document after completion. What was the lead's role title at that time?</question>
   <answer>Product Manager</answer>
</qa_pair>
```

这个问题好的原因：
- 没有使用特定的项目名称（"focused on improving customer onboarding"）
- 需要查找特定时间范围内已完成的项目
- 需要识别项目负责人及其角色
- 需要从回顾文档中理解上下文
- 答案是人类可读且稳定的
- 基于已完成的工作（不会变化）

**示例 3：需要多步骤的复杂聚合（Issue Tracker MCP）**
```xml
<qa_pair>
   <question>Among all bugs reported in January 2024 that were marked as critical priority, which assignee resolved the highest percentage of their assigned bugs within 48 hours? Provide the assignee's username.</question>
   <answer>alex_eng</answer>
</qa_pair>
```

这个问题好的原因：
- 需要按日期、优先级和状态过滤 bug
- 需要按负责人分组并计算解决率
- 需要理解时间戳以确定 48 小时窗口
- 测试分页（可能需要处理大量 bug）
- 答案是单一的用户名
- 基于特定时间段的历史数据

**示例 4：需要跨多种数据类型综合（CRM MCP）**
```xml
<qa_pair>
   <question>Find the account that upgraded from the Starter to Enterprise plan in Q4 2023 and had the highest annual contract value. What industry does this account operate in?</question>
   <answer>Healthcare</answer>
</qa_pair>
```

这个问题好的原因：
- 需要理解订阅层级变更
- 需要识别特定时间范围内的升级事件
- 需要比较合同价值
- 必须访问客户行业信息
- 答案简单且可验证
- 基于已完成的历史交易

### 不好的问题

**示例 1：答案随时间变化**
```xml
<qa_pair>
   <question>How many open issues are currently assigned to the engineering team?</question>
   <answer>47</answer>
</qa_pair>
```

这个问题不好的原因：
- 答案会随着 issue 的创建、关闭或重新分配而变化
- 不基于稳定/静态数据
- 依赖动态的"当前状态"

**示例 2：关键词搜索即可解决**
```xml
<qa_pair>
   <question>Find the pull request with title "Add authentication feature" and tell me who created it.</question>
   <answer>developer123</answer>
</qa_pair>
```

这个问题不好的原因：
- 可以通过精确标题的关键词搜索直接解决
- 不需要深入探索或理解
- 不需要综合或分析

**示例 3：答案格式模糊**
```xml
<qa_pair>
   <question>List all the repositories that have Python as their primary language.</question>
   <answer>repo1, repo2, repo3, data-pipeline, ml-tools</answer>
</qa_pair>
```

这个问题不好的原因：
- 答案是列表，可能以任何顺序返回
- 难以通过直接字符串比较验证
- LLM 可能以不同方式格式化（JSON 数组、逗号分隔、换行分隔）
- 更好的做法是询问特定的聚合值（计数）或极值（最多 star）

## 验证流程

创建评估后：

1. **检查 XML 文件**以了解 schema
2. **加载每个任务说明**，并行使用 MCP server 和 tool，通过自行尝试解决任务来确定正确答案
3. **标记任何需要写操作或破坏性操作**的任务
4. **汇总所有正确答案**并替换文档中的错误答案
5. **移除任何需要写操作或破坏性操作**的 `<qa_pair>`

记得并行化解决任务以避免上下文耗尽，然后汇总所有答案并在最后修改文件。

## 创建高质量评估的技巧

1. **深入思考并提前规划**再生成任务
2. **尽可能并行化**以加速流程和管理上下文
3. **关注真实用例**——人类真正想要完成的任务
4. **创建有挑战性的问题**来测试 MCP server 能力的极限
5. **确保稳定性**——使用历史数据和已关闭的概念
6. **验证答案**——使用 MCP server tool 自行解答问题
7. **迭代和改进**——基于在流程中学到的经验

---

# 运行评估

创建评估文件后，可以使用提供的评估工具来测试你的 MCP server。

## 设置

1. **安装依赖**

   ```bash
   pip install -r scripts/requirements.txt
   ```

   或手动安装：
   ```bash
   pip install anthropic mcp
   ```

2. **设置 API Key**

   ```bash
   export ANTHROPIC_API_KEY=your_api_key_here
   ```

## 评估文件格式

评估文件使用带有 `<qa_pair>` 元素的 XML 格式：

```xml
<evaluation>
   <qa_pair>
      <question>Find the project created in Q2 2024 with the highest number of completed tasks. What is the project name?</question>
      <answer>Website Redesign</answer>
   </qa_pair>
   <qa_pair>
      <question>Search for issues labeled as "bug" that were closed in March 2024. Which user closed the most issues? Provide their username.</question>
      <answer>sarah_dev</answer>
   </qa_pair>
</evaluation>
```

## 运行评估

评估脚本（`scripts/evaluation.py`）支持三种传输类型：

**重要说明：**
- **stdio 传输**：评估脚本会自动启动和管理 MCP server 进程。请勿手动运行 server。
- **sse/http 传输**：你必须在运行评估之前单独启动 MCP server。脚本连接到指定 URL 上已运行的 server。

### 1. 本地 STDIO Server

适用于本地运行的 MCP server（脚本自动启动 server）：

```bash
python scripts/evaluation.py \
  -t stdio \
  -c python \
  -a my_mcp_server.py \
  evaluation.xml
```

带环境变量：
```bash
python scripts/evaluation.py \
  -t stdio \
  -c python \
  -a my_mcp_server.py \
  -e API_KEY=abc123 \
  -e DEBUG=true \
  evaluation.xml
```

### 2. Server-Sent Events (SSE)

适用于基于 SSE 的 MCP server（你必须先启动 server）：

```bash
python scripts/evaluation.py \
  -t sse \
  -u https://example.com/mcp \
  -H "Authorization: Bearer token123" \
  -H "X-Custom-Header: value" \
  evaluation.xml
```

### 3. HTTP（Streamable HTTP）

适用于基于 HTTP 的 MCP server（你必须先启动 server）：

```bash
python scripts/evaluation.py \
  -t http \
  -u https://example.com/mcp \
  -H "Authorization: Bearer token123" \
  evaluation.xml
```

## 命令行选项

```
usage: evaluation.py [-h] [-t {stdio,sse,http}] [-m MODEL] [-c COMMAND]
                     [-a ARGS [ARGS ...]] [-e ENV [ENV ...]] [-u URL]
                     [-H HEADERS [HEADERS ...]] [-o OUTPUT]
                     eval_file

positional arguments:
  eval_file             Path to evaluation XML file

optional arguments:
  -h, --help            Show help message
  -t, --transport       Transport type: stdio, sse, or http (default: stdio)
  -m, --model           Claude model to use (default: claude-3-7-sonnet-20250219)
  -o, --output          Output file for report (default: print to stdout)

stdio options:
  -c, --command         Command to run MCP server (e.g., python, node)
  -a, --args            Arguments for the command (e.g., server.py)
  -e, --env             Environment variables in KEY=VALUE format

sse/http options:
  -u, --url             MCP server URL
  -H, --header          HTTP headers in 'Key: Value' format
```

## 输出

评估脚本生成详细报告，包括：

- **汇总统计**：
  - 准确率（正确/总数）
  - 平均任务耗时
  - 每个任务的平均 tool 调用次数
  - 总 tool 调用次数

- **每个任务的结果**：
  - Prompt 和预期响应
  - agent 的实际响应
  - 答案是否正确（✅/❌）
  - 耗时和 tool 调用详情
  - agent 对其方法的总结
  - agent 对 tool 的反馈

### 将报告保存到文件

```bash
python scripts/evaluation.py \
  -t stdio \
  -c python \
  -a my_server.py \
  -o evaluation_report.md \
  evaluation.xml
```

## 完整示例工作流

以下是创建和运行评估的完整示例：

1. **创建评估文件**（`my_evaluation.xml`）：

```xml
<evaluation>
   <qa_pair>
      <question>Find the user who created the most issues in January 2024. What is their username?</question>
      <answer>alice_developer</answer>
   </qa_pair>
   <qa_pair>
      <question>Among all pull requests merged in Q1 2024, which repository had the highest number? Provide the repository name.</question>
      <answer>backend-api</answer>
   </qa_pair>
   <qa_pair>
      <question>Find the project that was completed in December 2023 and had the longest duration from start to finish. How many days did it take?</question>
      <answer>127</answer>
   </qa_pair>
</evaluation>
```

2. **安装依赖**：

```bash
pip install -r scripts/requirements.txt
export ANTHROPIC_API_KEY=your_api_key
```

3. **运行评估**：

```bash
python scripts/evaluation.py \
  -t stdio \
  -c python \
  -a github_mcp_server.py \
  -e GITHUB_TOKEN=ghp_xxx \
  -o github_eval_report.md \
  my_evaluation.xml
```

4. **查看报告**（`github_eval_report.md`）以：
   - 查看哪些问题通过/失败
   - 阅读 agent 对你 tool 的反馈
   - 识别改进领域
   - 迭代优化你的 MCP server 设计

## 故障排除

### 连接错误

如果遇到连接错误：
- **STDIO**：验证命令和参数是否正确
- **SSE/HTTP**：检查 URL 是否可访问以及 header 是否正确
- 确保所有必需的 API key 已在环境变量或 header 中设置

### 准确率低

如果许多评估失败：
- 查看 agent 对每个任务的反馈
- 检查 tool 描述是否清晰全面
- 验证输入参数是否有完善的文档
- 考虑 tool 返回的数据是否过多或过少
- 确保错误消息具有可操作性

### 超时问题

如果任务超时：
- 使用更强大的模型（例如 `claude-3-7-sonnet-20250219`）
- 检查 tool 是否返回了过多数据
- 验证分页是否正常工作
- 考虑简化复杂问题
