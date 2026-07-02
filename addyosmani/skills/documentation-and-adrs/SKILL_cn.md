---
name: documentation-and-adrs
description: 记录决策和文档。在做架构决策、更改公共 API、发布功能，或需要记录供未来工程师和 agent 理解代码库的上下文时使用。
---

# 文档与 ADR

## 概述

记录决策，而不仅仅是代码。最有价值的文档捕捉的是*为什么*——导致某项决策的上下文、约束和权衡。代码展示*构建了什么*；文档解释*为什么要这样构建*以及*考虑过哪些替代方案*。这些上下文对于未来在代码库中工作的人类和 agent 至关重要。

## 何时使用

- 做出重大架构决策时
- 在多个竞争方案之间做选择时
- 添加或更改公共 API 时
- 发布会改变面向用户行为的功能时
- 让新团队成员（或 agent）上手项目时
- 当你发现自己在反复解释同一件事时

**何时不使用：** 不要为显而易见的代码写文档。不要添加复述代码已有内容的注释。不要为一次性原型写文档。

## 架构决策记录（ADR）

ADR 记录了重大技术决策背后的推理。这是你能写的最有价值的文档。

### 何时撰写 ADR

- 选择框架、库或主要依赖项
- 设计数据模型或数据库模式
- 选择认证策略
- 决定 API 架构（REST vs. GraphQL vs. tRPC）
- 在构建工具、托管平台或基础设施之间做选择
- 任何逆转成本高昂的决策

### ADR 模板

将 ADR 存放在 `docs/decisions/` 目录下，使用顺序编号：

```markdown
# ADR-001: 使用 PostgreSQL 作为主数据库

## 状态
Accepted | Superseded by ADR-XXX | Deprecated

## 日期
2025-01-15

## 上下文
我们需要为任务管理应用选择一个主数据库。核心需求：
- 关系型数据模型（用户、任务、团队之间有各种关联关系）
- 任务状态变更需要 ACID 事务
- 支持任务内容的全文搜索
- 需要提供托管服务（团队小，运维能力有限）

## 决策
使用 PostgreSQL，搭配 Prisma ORM。

## 考虑的替代方案

### MongoDB
- 优点：灵活的模式，上手容易
- 缺点：我们的数据本质上是关系型的；需要手动管理关系
- 驳回理由：把关系型数据放在文档存储中会导致复杂的连接或数据重复

### SQLite
- 优点：零配置，嵌入式，读取速度快
- 缺点：并发写入支持有限，没有适合生产的托管服务
- 驳回理由：不适合生产环境中的多用户 Web 应用

### MySQL
- 优点：成熟，广泛支持
- 缺点：PostgreSQL 在 JSON 支持、全文搜索和生态工具链方面更优
- 驳回理由：PostgreSQL 更符合我们的功能需求

## 后果
- Prisma 提供类型安全的数据库访问和迁移管理
- 可以使用 PostgreSQL 的全文搜索，而无需额外引入 Elasticsearch
- 团队需要 PostelSQL 知识（常规技能，风险低）
- 使用托管服务（Supabase、Neon 或 RDS）
```

### ADR 生命周期

```
PROPOSED → ACCEPTED → (SUPERSEDED or DEPRECATED)
```

- **不要删除旧的 ADR。** 它们承载历史上下文。
- 当决策变更时，撰写新的 ADR，引用并取代旧的 ADR。

## 内联文档

### 何时加注释

注释*为什么*，而不是*是什么*：

```typescript
// BAD: 复述代码
// 将计数器加 1
counter += 1;

// GOOD: 解释非显而易见的意图
// 速率限制使用滑动窗口——在窗口边界重置计数器，
// 而非按固定时间表，以防止窗口边缘的突发攻击
if (now - windowStart > WINDOW_SIZE_MS) {
  counter = 0;
  windowStart = now;
}
```

### 何时不加注释

```typescript
// 不要为自解释的代码加注释
function calculateTotal(items: CartItem[]): number {
  return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
}

// 不要留下 TODO 注释来代替现在就应该做的事情
// TODO: 添加错误处理  ← 直接加上

// 不要留下被注释掉的代码
// const oldImplementation = () => { ... }  ← 删掉它，git 有历史记录
```

### 记录已知陷阱

```typescript
/**
 * IMPORTANT: 此函数必须在首次渲染之前调用。
 * 如果在 hydration 之后调用，会导致未设置样式的闪烁，
 * 因为主题上下文在 SSR 期间不可用。
 *
 * 参见 ADR-003 了解完整的设计理由。
 */
export function initializeTheme(theme: Theme): void {
  // ...
}
```

## API 文档

对于公共 API（REST、GraphQL、库接口）：

### 内联类型（TypeScript 首选）

```typescript
/**
 * 创建新任务。
 *
 * @param input - 任务创建数据（标题必填，描述可选）
 * @returns 创建的任务，包含服务端生成的 ID 和时间戳
 * @throws {ValidationError} 如果标题为空或超过 200 个字符
 * @throws {AuthenticationError} 如果用户未认证
 *
 * @example
 * const task = await createTask({ title: '购买杂货' });
 * console.log(task.id); // "task_abc123"
 */
export async function createTask(input: CreateTaskInput): Promise<Task> {
  // ...
}
```

### REST API 的 OpenAPI / Swagger

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
          description: 验证错误
```

## README 结构

每个项目都应有一个 README，涵盖以下内容：

```markdown
# 项目名称

一段话描述这个项目是做什么的。

## 快速开始
1. 克隆仓库
2. 安装依赖: `npm install`
3. 设置环境: `cp .env.example .env`
4. 启动开发服务器: `npm run dev`

## 命令
| 命令 | 描述 |
|---------|-------------|
| `npm run dev` | 启动开发服务器 |
| `npm test` | 运行测试 |
| `npm run build` | 生产构建 |
| `npm run lint` | 运行 linter |

## 架构
简要概述项目结构和关键设计决策。
链接到 ADR 以获取详细信息。

## 贡献
如何贡献、编码标准、PR 流程。
```

## Changelog 维护

对于已发布的功能：

```markdown
# Changelog

## [1.2.0] - 2025-01-20
### Added
- 任务分享：用户可以分享任务给团队成员（#123）
- 任务分配的邮件通知（#124）

### Fixed
- 快速点击创建按钮时出现重复任务（#125）

### Changed
- 任务列表现在每页加载 50 条（原为 20 条），以改善用户体验（#126）
```

## 面向 Agent 的文档

为 AI agent 上下文做特殊考虑：

- **CLAUDE.md / rules 文件** — 文档化项目约定，使 agent 能够遵循
- **Spec 文件** — 保持 spec 更新，使 agent 构建正确的内容
- **ADR** — 帮助 agent 理解为何做出过去的决策（防止重复决策）
- **内联陷阱** — 防止 agent 掉入已知的坑

## 常见借口

| 借口 | 现实 |
|---|---|
| "代码是自文档化的" | 代码展示是什么。它不展示为什么、拒绝了哪些替代方案、适用哪些约束。 |
| "等 API 稳定了再写文档" | 当你写文档时，API 会更快稳定。文档是设计的首次测试。 |
| "没人读文档" | Agent 会读。未来的工程师会读。三个月后的你自己会读。 |
| "ADR 是额外负担" | 10 分钟的 ADR 可以避免六个月后对同一个决策的 2 小时争论。 |
| "注释会过时" | 关于*为什么*的注释是稳定的。关于*是什么*的注释会过时——这就是你只写前者的原因。 |

## 警示信号

- 架构决策没有书面理由
- 公共 API 没有文档或类型
- README 没有解释如何运行项目
- 包含被注释掉的代码，而不是删除掉
- TODO 注释已经存在数周
- 项目中有重大架构选择却没有 ADR
- 文档复述代码而不是解释意图

## 验证

完成文档后：

- [ ] 所有重大架构决策都有对应的 ADR
- [ ] README 涵盖快速开始、命令和架构概述
- [ ] API 函数有参数和返回类型文档
- [ ] 已知陷阱在相关位置有内联文档
- [ ] 没有残留的被注释掉的代码
- [ ] Rules 文件（CLAUDE.md 等）是最新和准确的
