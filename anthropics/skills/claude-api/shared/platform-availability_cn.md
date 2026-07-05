# 平台可用性

各功能在哪些提供商平台上可用。**此表格是本技能中的唯一权威来源**——其他按功能分节的说明均指向此处，而非重复说明可用性。在为第三方平台（Bedrock、Vertex、Foundry）或 Claude Platform on AWS 编写代码时，请先查阅此表格；如果某功能在该平台上不受支持，则需使用第一方 Claude API 接口或其他方案。

列说明：**1P** = 第一方 Claude API，**P-AWS** = Claude Platform on AWS（Anthropic 运营，当日同步），**Bedrock** = Amazon Bedrock，**Vertex** = Google Cloud Vertex AI，**Foundry** = Microsoft Foundry。✅ = 正式发布（GA），β = beta，❌ = 不支持。

| 功能 | 1P | P-AWS | Bedrock | Vertex | Foundry | 备注 |
|---|---|---|---|---|---|---|
| Messages、流式传输、tool use | ✅ | ✅ | ✅ | ✅ | ✅ | 核心 API |
| PDF 输入 | ✅ | ✅ | ✅ | ✅ | β | |
| 结构化输出 / 严格 tool use | ✅ | ✅ | ✅ | ✅ | β | |
| Adaptive thinking / effort | ✅ | ✅ | ✅ | ✅ | β | |
| Extended thinking | ✅ | ✅ | ✅ | ✅ | β | |
| Prompt 缓存（5 分钟、1 小时） | ✅ | ✅ | ✅ | ✅ | β | |
| 自动 prompt 缓存 | ✅ | ✅ | ❌ | ❌ | β | |
| Token 计数 | ✅ | ✅ | ✅ | ✅ | β | |
| 引用 | ✅ | ✅ | ✅ | ✅ | β | |
| 搜索结果内容块 | ✅ | ✅ | ✅ | ✅ | β | |
| 细粒度 tool 流式传输 | ✅ | ✅ | ✅ | ✅ | ✅ | |
| 压缩（Compaction） | β | β | β | β | β | |
| 上下文编辑 | β | β | β | β | β | |
| 上下文窗口（1M） | ✅ | ✅ | ✅ | ✅ | β | |
| `inference_geo`（数据驻留） | ✅ | ✅ | ❌ | ❌ | ❌ | |
| **服务端工具** | | | | | | |
| &nbsp;&nbsp;Web 搜索 | ✅ | ✅ | ❌ | ✅ | β | Vertex：仅支持基础 `web_search_20250305`（不支持 `_20260209` 动态过滤） |
| &nbsp;&nbsp;Web 获取 | ✅ | ✅ | ❌ | ❌ | β | |
| &nbsp;&nbsp;代码执行 | ✅ | ✅ | ❌ | ❌ | β | |
| &nbsp;&nbsp;Tool 搜索 | ✅ | ✅ | ✅ | ✅ | β | Bedrock：仅 InvokeModel API，不支持 Converse |
| &nbsp;&nbsp;Advisor 工具 | β | β | ❌ | ❌ | ❌ | |
| **客户端实现的工具** | | | | | | |
| &nbsp;&nbsp;Bash、文本编辑器、记忆 | ✅ | ✅ | ✅ | ✅ | β | |
| &nbsp;&nbsp;Computer use | β | β | β | β | β | |
| **Agentic / 编排** | | | | | | |
| &nbsp;&nbsp;Agent Skills（Messages API） | β | β | ❌ | ❌ | β | |
| &nbsp;&nbsp;程序化 tool 调用 | ✅ | ✅ | ❌ | ❌ | β | |
| &nbsp;&nbsp;MCP 连接器 | β | β | ❌ | ❌ | β | |
| &nbsp;&nbsp;托管 Agents | β | β | ❌ | ❌ | ❌ | Foundry ❌ 为推断（Foundry 文档中同样未提及） |
| &nbsp;&nbsp;自托管沙箱 | β | β | ❌ | ❌ | ❌ | P-AWS：不支持 `GET /v1/environments/{id}/work` 列表端点；其他 work 端点正常 |
| **API 端点** | | | | | | |
| &nbsp;&nbsp;Message Batches | ✅ | ✅ | ❌ | ❌ | ❌ | |
| &nbsp;&nbsp;Files API | β | β | ❌ | ❌ | β | |
| &nbsp;&nbsp;Models API | ✅ | ✅ | ❌ | ❌ | ❌ | |
| **其他** | | | | | | |
| &nbsp;&nbsp;会话中途 system 消息 | ✅ | ✅ | ❌ | ❌ | ❌ | 仅限 Claude Opus 4.8 |
| &nbsp;&nbsp;快速模式 | β | ❌ | ❌ | ❌ | ❌ | 研究预览，beta `fast-mode-2026-02-01`，仅限第一方 API |
| &nbsp;&nbsp;缓存诊断 | β | ❌ | ❌ | ❌ | ❌ | 仅限第一方 API |
| &nbsp;&nbsp;任务预算 | β | β | ❌ | ❌ | ❌ | Beta header `task-budgets-2026-03-13`；第三方可用性未文档化——视为不支持 |

<!--
GROUNDING（仅审查用；运行时由 processSkillMarkdown 剥离）。
所有路径位于 docker_eval/resources/cdp-skill/public-docs/ 下。

主要来源：build-with-claude/overview.mdx <PlatformAvailability> props
（claudeApi→1P, claudePlatformAws→P-AWS, bedrock→Bedrock, vertexAi→Vertex,
azureAi→Foundry; *Beta 后缀→β; 属性缺失→❌）。各行引用：

  Context windows          ov:44
  Adaptive thinking        ov:45
  Batch / Message Batches  ov:46; bed:360; vtx:381; fdy:507
  Citations                ov:47
  inference_geo            ov:48
  Effort                   ov:49
  Extended thinking        ov:50
  PDF input                ov:51
  Search results           ov:52
  Structured outputs       ov:53
  Advisor tool             ov:63
  Code execution           ov:64
  Web fetch                ov:65
  Web search               ov:66; agents-and-tools/tool-use/web-search-tool.mdx:41
  Bash/text-editor/memory  ov:72,75,74
  Computer use             ov:73
  Agent Skills             ov:83
  Fine-grained streaming   ov:84
  MCP connector            ov:85; agents-and-tools/mcp-connector.mdx:36
  Programmatic tool call   ov:86
  Tool search              ov:87; agents-and-tools/tool-use/tool-search-tool.mdx:24-30
  Compaction               ov:95
  Context editing          ov:96
  Automatic caching        ov:97
  Prompt caching 5m/1h     ov:98,99
  Token counting           ov:100
  Files API                ov:108; build-with-claude/files.mdx:17
  Managed Agents           managed-agents/overview.mdx:11,70-72; bed:360; vtx:381
  Self-hosted sandboxes    build-with-claude/claude-platform-on-aws.mdx:525,547
  Mid-convo system msgs    build-with-claude/mid-conversation-system-messages.mdx:15
  Fast mode                build-with-claude/fast-mode.mdx:23
  Cache diagnostics        build-with-claude/cache-diagnostics.mdx:15,1379
  Task budgets             build-with-claude/task-budgets.mdx:15
  Models API               bed:360; vtx:381; fdy:506

  ov  = build-with-claude/overview.mdx
  bed = build-with-claude/claude-in-amazon-bedrock.mdx
  vtx = build-with-claude/claude-on-vertex-ai.mdx
  fdy = build-with-claude/claude-in-microsoft-foundry.mdx
 -->