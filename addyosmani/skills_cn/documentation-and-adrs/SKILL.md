---
name: documentation-and-adrs
description: 记录决策与文档。适用于进行架构决策、变更公共 API、发布功能，或需要记录未来工程师和 AI 代理理解代码库所需上下文的场景。
---

# 文档与 ADR（架构决策记录）

## 概述

记录决策，而不仅仅是代码。最有价值的文档记录的是 *为什么* —— 导致某个决策的上下文、约束和权衡。代码展示的是 *做了什么*；文档解释的是 *为什么这样做* 以及 *考虑过哪些替代方案*。这些上下文对于未来在代码库中工作的人类和 AI 代理至关重要。

## 何时使用

- 做出重大架构决策时
- 在多个竞争方案之间进行选择时
- 添加或变更公共 API 时
- 发布改变用户可见行为的功能时
- 为新团队成员（或 AI 代理）进行项目入门时
- 当你发现自己在反复解释同一件事时

**何时不使用：** 不要为显而易见的代码写文档。不要添加重复代码已表达内容的注释。不要为一次性原型写文档。

## 架构决策记录（ADR）

ADR 记录重大技术决策背后的推理过程。它们是你能写的最高价值的文档。

### 何时撰写 ADR

- 选择框架、库或主要依赖时
- 设计数据模型或数据库模式时
- 选择认证策略时
- 决定 API 架构时（REST vs. GraphQL vs. tRPC）
- 在构建工具、托管平台或基础设施之间选择时
- 任何回退代价高昂的决策

### ADR 模板

将 ADR 存储在 `docs/decisions/` 目录下，并按顺序编号：

```markdown
# ADR-001: 使用 PostgreSQL 作为主数据库

## 状态
已接受 | 已被 ADR-XXX 取代 | 已废弃

## 日期
2025-01-15

## 上下文
我们需要为任务管理应用选择一个主数据库。关键需求：
- 关系型数据模型（用户、任务、团队及其关联关系）
- 任务状态变更需要 ACID 事务支持
- 支持对任务内容进行全文搜索
- 提供托管服务（适用于小团队，运维能力有限）

## 决策
使用 PostgreSQL 配合 Prisma ORM。

## 已考虑的替代方案

### MongoDB
- 优点：灵活的 Schema，上手容易
- 缺点：我们的数据本质上是关系型的；需要手动管理关联关系
- 否决原因：将关系型数据存储在文档数据库中会导致复杂的联表查询或数据冗余

### SQLite
- 优点：零配置，嵌入式，读取速度快
- 缺点：并发写入支持有限，生产环境无托管服务
- 否决原因：不适合生产环境中的多用户 Web 应用

### MySQL
- 优点：成熟，广泛支持
- 缺点：PostgreSQL 在 JSON 支持、全文搜索和生态工具方面更优
- 否决原因：PostgreSQL 更符合我们的功能需求

## 影响
- Prisma 提供类型安全的数据库访问和迁移管理
- 我们可以使用 PostgreSQL 的全文搜索，而无需引入 Elasticsearch
- 团队需要具备 PostgreSQL 知识（通用技能，风险低）
- 使用托管服务部署（Supabase、Neon 或 RDS）
```

### ADR 生命周期

```
提议中 → 已接受 → (已取代 或 已废弃)
```

- **不要删除旧的 ADR。** 它们记录了历史上下文。
- 当决策发生变更时，撰写新的 ADR 来引用并取代旧的 ADR。

## 行内文档

### 何时添加注释

注释应解释 *为什么*，而不是 *做了什么*：

```typescript
// 错误：重复代码已表达的内容
// 将计数器加 1
counter += 1;

// 正确：解释不明显的意图
// 速率限制使用滑动窗口 —— 在窗口边界重置计数器，
// 而非按固定周期重置，以防止窗口边缘的突发攻击
if (now - windowStart > WINDOW_SIZE_MS) {
  counter = 0;
  windowStart = now;
}
```

### 何时不添加注释

```typescript
// 不要为自解释的代码添加注释
function calculateTotal(items: CartItem[]): number {
  return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
}

// 不要为应该立即完成的事项留下 TODO 注释
// TODO: add error handling  ← 现在就加上

// 不要留下被注释掉的代码
// const oldImplementation = () => { ... }  ← 删除它，git 有历史记录
```

### 记录已知的注意事项

```typescript
/**
 * 重要：此函数必须在首次渲染之前调用。
 * 如果在 hydration 之后调用，会导致无样式内容闪烁，
 * 因为 SSR 期间主题上下文不可用。
 *
 * 完整的设计原理参见 ADR-003。
 */
export function initializeTheme(theme: Theme): void {
  // ...
}
```

## API 文档

针对公共 API（REST、GraphQL、库接口）：

### 与类型定义内联（TypeScript 推荐方式）

```typescript
/**
 * 创建一个新任务。
 *
 * @param input - 任务创建数据（title 必填，description 可选）
 * @returns 创建的任务，包含服务端生成的 ID 和时间戳
 * @throws {ValidationError} 当 title 为空或超过 200 个字符时
 * @throws {AuthenticationError} 当用户未认证时
 *
 * @example
 * const task = await createTask({ title: 'Buy groceries' });
 * console.log(task.id); // "task_abc123"
 */
export async function createTask(input: CreateTaskInput): Promise<Task> {
  // ...
}
```

### 使用 OpenAPI / Swagger 描述 REST API

```yaml
paths:
  /api/tasks:
    post:
      summary: 创建任务
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateTaskInput'
      responses:
        '201':
          description: 任务已创建
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Task'
        '422':
          description: 校验错误
```

## README 结构

每个项目都应有一个 README，涵盖以下内容：

```markdown
# 项目名称

一段话描述此项目的功能。

## 快速开始
1. 克隆仓库
2. 安装依赖：`npm install`
3. 配置环境变量：`cp .env.example .env`
4. 启动开发服务器：`npm run dev`

## 命令
| 命令 | 描述 |
|------|------|
| `npm run dev` | 启动开发服务器 |
| `npm test` | 运行测试 |
| `npm run build` | 生产环境构建 |
| `npm run lint` | 运行代码检查 |

## 架构
项目结构和关键设计决策的简要概述。
详细信息参见 ADR。

## 贡献指南
如何贡献、编码规范、PR 流程。
```

## 变更日志维护

针对已发布的功能：

```markdown
# 变更日志

## [1.2.0] - 2025-01-20
### 新增
- 任务共享：用户可以与团队成员共享任务 (#123)
- 任务分配的邮件通知 (#124)

### 修复
- 快速点击创建按钮时出现重复任务的问题 (#125)

### 变更
- 任务列表现在每页加载 50 条（原为 20 条），以提升用户体验 (#126)
```

## 面向 AI 代理的文档

针对 AI 代理场景的特殊考量：

- **CLAUDE.md / 规则文件** —— 记录项目约定，使 AI 代理遵循它们
- **规格文件** —— 保持规格更新，使 AI 代理构建正确的功能
- **ADR** —— 帮助 AI 代理理解过去决策的原因（防止重复决策）
- **行内注意事项** —— 防止 AI 代理陷入已知的陷阱

## 常见的自我合理化

| 合理化说法 | 现实 |
|---|---|
| "代码本身就是文档" | 代码展示的是做了什么，而不是为什么、否决了哪些替代方案、存在哪些约束。 |
| "等 API 稳定了再写文档" | 写文档反而能加速 API 的稳定。文档是设计的第一个验证。 |
| "没人看文档" | AI 代理会看。未来的工程师会看。三个月后的你自己也会看。 |
| "写 ADR 是额外负担" | 10 分钟的 ADR 可以避免六个月后关于同一决策的 2 小时争论。 |
| "注释会过时" | 关于 *为什么* 的注释是稳定的。关于 *做了什么* 的注释会过时 —— 这就是你只应写前者的原因。 |

## 危险信号

- 架构决策没有书面理由
- 公共 API 没有文档或类型定义
- README 没有说明如何运行项目
- 用注释掉的代码代替删除
- 存在数周的 TODO 注释
- 有重大架构选择的项目没有任何 ADR
- 文档只是复述代码，而非解释意图

## 验证清单

完成文档后：

- [ ] 所有重大架构决策都有对应的 ADR
- [ ] README 涵盖快速开始、命令和架构概述
- [ ] API 函数具有参数和返回类型的文档
- [ ] 已知的注意事项在关键位置以行内方式记录
- [ ] 没有残留的注释代码
- [ ] 规则文件（CLAUDE.md 等）是最新且准确的
