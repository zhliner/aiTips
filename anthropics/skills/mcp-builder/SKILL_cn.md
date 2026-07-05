---
name: mcp-builder
description: 创建高质量 MCP（Model Context Protocol）server 的指南，使 LLM 能够通过精心设计的 tool 与外部服务交互。适用于构建 MCP server 以集成外部 API 或服务的场景，支持 Python（FastMCP）和 Node/TypeScript（MCP SDK）。
license: Complete terms in LICENSE.txt
---

# MCP Server 开发指南

## 概述

创建 MCP（Model Context Protocol）server，使 LLM 能够通过精心设计的 tool 与外部服务交互。MCP server 的质量取决于它在多大程度上帮助 LLM 完成真实世界的任务。

---

# 流程

## 🚀 高层工作流

创建一个高质量的 MCP server 涉及四个主要阶段：

### 阶段 1：深入调研与规划

#### 1.1 理解现代 MCP 设计

**API 覆盖 vs. 工作流 tool：**
在全面的 API endpoint 覆盖与专门的工作流 tool 之间取得平衡。工作流 tool 在特定任务中可能更加便捷，而全面的覆盖则赋予 agent 灵活组合操作的能力。性能因 client 而异——某些 client 受益于代码执行来组合基础 tool，而另一些则更适合高层工作流。在不确定时，优先保证全面的 API 覆盖。

**Tool 命名与可发现性：**
清晰、描述性的 tool 名称有助于 agent 快速找到合适的 tool。使用一致的前缀（例如 `github_create_issue`、`github_list_repos`）和面向动作的命名方式。

**上下文管理：**
agent 受益于简洁的 tool 描述以及过滤/分页结果的能力。设计返回聚焦、相关数据的 tool。某些 client 支持代码执行，可以帮助 agent 高效过滤和处理数据。

**可操作的错误消息：**
错误消息应通过具体的建议和后续步骤引导 agent 找到解决方案。

#### 1.2 研读 MCP 协议文档

**浏览 MCP 规范：**

从 sitemap 开始查找相关页面：`https://modelcontextprotocol.io/sitemap.xml`

然后使用 `.md` 后缀获取特定页面的 Markdown 格式内容（例如 `https://modelcontextprotocol.io/specification/draft.md`）。

需要重点查看的页面：
- 规范概述与架构
- 传输机制（streamable HTTP、stdio）
- Tool、resource 和 prompt 定义

#### 1.3 研读框架文档

**推荐技术栈：**
- **语言**：TypeScript（高质量的 SDK 支持以及在许多执行环境如 MCPB 中的良好兼容性。此外，AI 模型擅长生成 TypeScript 代码，受益于其广泛的使用、静态类型和良好的 lint 工具）
- **传输**：远程 server 使用 streamable HTTP，采用无状态 JSON（相比有状态 session 和流式响应，更易于扩展和维护）。本地 server 使用 stdio。

**加载框架文档：**

- **MCP 最佳实践**：[📋 查看最佳实践](./reference/mcp_best_practices.md) - 核心指南

**TypeScript（推荐）：**
- **TypeScript SDK**：使用 WebFetch 加载 `https://raw.githubusercontent.com/modelcontextprotocol/typescript-sdk/main/README.md`
- [⚡ TypeScript 指南](./reference/node_mcp_server.md) - TypeScript 模式与示例

**Python：**
- **Python SDK**：使用 WebFetch 加载 `https://raw.githubusercontent.com/modelcontextprotocol/python-sdk/main/README.md`
- [🐍 Python 指南](./reference/python_mcp_server.md) - Python 模式与示例

#### 1.4 规划实现

**了解 API：**
查阅服务的 API 文档，识别关键 endpoint、身份验证要求和数据模型。根据需要结合使用 web 搜索和 WebFetch。

**Tool 选择：**
优先保证全面的 API 覆盖。列出要实现 endpoint，从最常见的操作开始。

---

### 阶段 2：实现

#### 2.1 搭建项目结构

请参阅语言特定指南了解项目搭建：
- [⚡ TypeScript 指南](./reference/node_mcp_server.md) - 项目结构、package.json、tsconfig.json
- [🐍 Python 指南](./reference/python_mcp_server.md) - 模块组织、依赖管理

#### 2.2 实现核心基础设施

创建共享工具：
- 带身份验证的 API client
- 错误处理辅助函数
- 响应格式化（JSON/Markdown）
- 分页支持

#### 2.3 实现 Tool

每个 tool 需要：

**输入 Schema：**
- 使用 Zod（TypeScript）或 Pydantic（Python）
- 包含约束条件和清晰的描述
- 在字段描述中添加示例

**输出 Schema：**
- 尽可能定义 `outputSchema` 以提供结构化数据
- 在 tool 响应中使用 `structuredContent`（TypeScript SDK 特性）
- 帮助 client 理解和处理 tool 输出

**Tool 描述：**
- 功能的简洁概述
- 参数描述
- 返回类型 schema

**实现：**
- 对 I/O 操作使用 async/await
- 提供带有可操作消息的适当错误处理
- 在适用场景下支持分页
- 使用现代 SDK 时同时返回文本内容和结构化数据

**注解（Annotations）：**
- `readOnlyHint`：true/false
- `destructiveHint`：true/false
- `idempotentHint`：true/false
- `openWorldHint`：true/false

---

### 阶段 3：审查与测试

#### 3.1 代码质量

审查要点：
- 无重复代码（DRY 原则）
- 一致的错误处理
- 完整的类型覆盖
- 清晰的 tool 描述

#### 3.2 构建与测试

**TypeScript：**
- 运行 `npm run build` 验证编译
- 使用 MCP Inspector 测试：`npx @modelcontextprotocol/inspector`

**Python：**
- 验证语法：`python -m py_compile your_server.py`
- 使用 MCP Inspector 测试

请参阅语言特定指南了解详细的测试方法和质量检查清单。

---

### 阶段 4：创建评估

实现 MCP server 后，创建全面的评估来测试其有效性。

**加载 [✅ 评估指南](./reference/evaluation.md) 获取完整的评估指南。**

#### 4.1 理解评估目的

使用评估来测试 LLM 是否能有效使用你的 MCP server 回答真实的、复杂的问题。

#### 4.2 创建 10 道评估问题

要创建有效的评估，请遵循评估指南中概述的流程：

1. **Tool 检查**：列出可用的 tool 并了解其功能
2. **内容探索**：使用只读操作探索可用数据
3. **问题生成**：创建 10 道复杂的、真实的问题
4. **答案验证**：自行解答每道问题以验证答案

#### 4.3 评估要求

确保每道问题满足：
- **独立性**：不依赖于其他问题
- **只读**：仅需非破坏性操作
- **复杂性**：需要多次 tool 调用和深入探索
- **真实性**：基于人类真正关心的真实用例
- **可验证性**：具有单一的、可通过字符串比较验证的明确答案
- **稳定性**：答案不会随时间变化

#### 4.4 输出格式

创建一个具有以下结构的 XML 文件：

```xml
<evaluation>
  <qa_pair>
    <question>Find discussions about AI model launches with animal codenames. One model needed a specific safety designation that uses the format ASL-X. What number X was being determined for the model named after a spotted wild cat?</question>
    <answer>3</answer>
  </qa_pair>
<!-- More qa_pairs... -->
</evaluation>
```

---

# 参考文件

## 📚 文档库

在开发过程中根据需要加载这些资源：

### 核心 MCP 文档（优先加载）
- **MCP 协议**：从 sitemap 开始 `https://modelcontextprotocol.io/sitemap.xml`，然后使用 `.md` 后缀获取特定页面
- [📋 MCP 最佳实践](./reference/mcp_best_practices.md) - 通用 MCP 指南，包括：
  - Server 和 tool 命名规范
  - 响应格式指南（JSON vs Markdown）
  - 分页最佳实践
  - 传输方式选择（streamable HTTP vs stdio）
  - 安全和错误处理标准

### SDK 文档（在阶段 1/2 加载）
- **Python SDK**：从 `https://raw.githubusercontent.com/modelcontextprotocol/python-sdk/main/README.md` 获取
- **TypeScript SDK**：从 `https://raw.githubusercontent.com/modelcontextprotocol/typescript-sdk/main/README.md` 获取

### 语言特定实现指南（在阶段 2 加载）
- [🐍 Python 实现指南](./reference/python_mcp_server.md) - 完整的 Python/FastMCP 指南，包含：
  - Server 初始化模式
  - Pydantic 模型示例
  - 使用 `@mcp.tool` 注册 tool
  - 完整的可运行示例
  - 质量检查清单

- [⚡ TypeScript 实现指南](./reference/node_mcp_server.md) - 完整的 TypeScript 指南，包含：
  - 项目结构
  - Zod schema 模式
  - 使用 `server.registerTool` 注册 tool
  - 完整的可运行示例
  - 质量检查清单

### 评估指南（在阶段 4 加载）
- [✅ 评估指南](./reference/evaluation.md) - 完整的评估创建指南，包含：
  - 问题创建指南
  - 答案验证策略
  - XML 格式规范
  - 示例问题和答案
  - 使用提供的脚本运行评估
