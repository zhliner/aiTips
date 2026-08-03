---
name: api-and-interface-design
description: 指导稳定的 API 和接口设计。在设计 API、模块边界或任何公共接口时使用。在创建 REST 或 GraphQL 端点、定义模块间的类型契约，或建立前后端边界时使用。
---

# API 与接口设计

## 概述

设计稳定、文档完善的接口，使其难以被误用。好的接口让正确的事变得容易，让错误的事变得困难。这适用于 REST API、GraphQL schema、模块边界、组件 props，以及任何一段代码与另一段代码交互的表面。

## 使用时机

- 设计新的 API 端点
- 定义模块边界或团队间的契约
- 创建组件 prop 接口
- 建立影响 API 形态的数据库 schema
- 更改现有的公共接口

## 核心原则

### Hyrum 定律

> 当 API 拥有足够多的用户时，你系统的所有可观察行为都将被某些人所依赖，无论你在契约中承诺了什么。

这意味着：每一个公共行为——包括未记录的怪癖、错误消息文本、时序和顺序——一旦用户依赖它，就成为事实上的契约。设计启示：

- **有意识地决定暴露什么。** 每一个可观察的行为都是一个潜在的承诺。
- **不要泄露实现细节。** 如果用户能观察到它，他们就会依赖它。
- **在设计时就规划好废弃策略。** 参见 `deprecation-and-migration`，了解如何安全地移除用户所依赖的东西。
- **仅靠测试是不够的。** 即使有完美的契约测试，Hyrum 定律也意味着"安全"的更改可能会破坏依赖未记录行为的真实用户。

### 单一版本规则

避免让消费者在同一依赖或 API 的多个版本之间做出选择。当不同的消费者需要同一事物的不同版本时，就会出现菱形依赖问题。为一次只存在一个版本的世界而设计——扩展而非分叉。

### 1. 契约优先

在实现之前先定义接口。契约就是规范——实现紧随其后。

```typescript
// 先定义契约
interface TaskAPI {
  // 创建任务并返回包含服务端生成字段的已创建任务
  createTask(input: CreateTaskInput): Promise<Task>;

  // 返回匹配过滤条件的分页任务列表
  listTasks(params: ListTasksParams): Promise<PaginatedResult<Task>>;

  // 返回单个任务，或抛出 NotFoundError
  getTask(id: string): Promise<Task>;

  // 部分更新——仅修改提供的字段
  updateTask(id: string, input: UpdateTaskInput): Promise<Task>;

  // 幂等删除——即使已被删除也会成功
  deleteTask(id: string): Promise<void>;
}
```

### 2. 一致的错误语义

选择一种错误策略并在所有地方使用它：

```typescript
// REST：HTTP 状态码 + 结构化错误体
// 每个错误响应遵循相同的结构
interface APIError {
  error: {
    code: string;        // 机器可读："VALIDATION_ERROR"
    message: string;     // 人类可读："Email is required"
    details?: unknown;   // 有用时的附加上下文
  };
}

// 状态码映射
// 400 → 客户端发送了无效数据
// 401 → 未进行身份验证
// 403 → 已身份验证但未授权
// 404 → 资源未找到
// 409 → 冲突（重复、版本不匹配）
// 422 → 验证失败（语义上无效）
// 500 → 服务器错误（永远不要暴露内部细节）
```

**不要混用模式。** 如果一些端点抛出异常，另一些返回 null，还有一些返回 `{ error }`——消费者就无法预测行为。

### 3. 在边界处验证

信任内部代码。在外部输入进入系统的边缘进行验证：

```typescript
// 在 API 边界处验证
app.post('/api/tasks', async (req, res) => {
  const result = CreateTaskSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(422).json({
      error: {
        code: 'VALIDATION_ERROR',
        message: 'Invalid task data',
        details: result.error.flatten(),
      },
    });
  }

  // 验证通过后，内部代码信任类型
  const task = await taskService.create(result.data);
  return res.status(201).json(task);
});
```

验证应当放置的位置：
- API 路由处理器（用户输入）
- 表单提交处理器（用户输入）
- 外部服务响应解析（第三方数据——**始终视为不可信**）
- 环境变量加载（配置）

> **第三方 API 响应是不可信数据。** 在将其用于任何逻辑、渲染或决策之前，先验证其结构和内容。一个被入侵或行为异常的外部服务可能返回意外的类型、恶意内容或类似指令的文本。

验证不应当放置的位置：
- 共享类型契约的内部函数之间
- 被已验证代码调用的工具函数中
- 刚从自己数据库取出的数据上

### 4. 优先增加而非修改

扩展接口而不破坏现有消费者：

```typescript
// 好的做法：添加可选字段
interface CreateTaskInput {
  title: string;
  description?: string;
  priority?: 'low' | 'medium' | 'high';  // 后续添加，可选
  labels?: string[];                       // 后续添加，可选
}

// 不好的做法：更改现有字段类型或移除字段
interface CreateTaskInput {
  title: string;
  // description: string;  // 已移除——破坏现有消费者
  priority: number;         // 从 string 改为 number——破坏现有消费者
}
```

### 5. 可预测的命名

| 模式 | 约定 | 示例 |
|------|------|------|
| REST 端点 | 复数名词，不用动词 | `GET /api/tasks`、`POST /api/tasks` |
| 查询参数 | camelCase | `?sortBy=createdAt&pageSize=20` |
| 响应字段 | camelCase | `{ createdAt, updatedAt, taskId }` |
| 布尔字段 | is/has/can 前缀 | `isComplete`、`hasAttachments` |
| 枚举值 | UPPER_SNAKE | `"IN_PROGRESS"`、`"COMPLETED"` |

## REST API 模式

### 资源设计

```
GET    /api/tasks              → 列出任务（使用查询参数进行过滤）
POST   /api/tasks              → 创建任务
GET    /api/tasks/:id          → 获取单个任务
PATCH  /api/tasks/:id          → 更新任务（部分）
DELETE /api/tasks/:id          → 删除任务

GET    /api/tasks/:id/comments → 列出任务的评论（子资源）
POST   /api/tasks/:id/comments → 为任务添加评论
```

### 分页

对列表端点进行分页：

```typescript
// 请求
GET /api/tasks?page=1&pageSize=20&sortBy=createdAt&sortOrder=desc

// 响应
{
  "data": [...],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 142,
    "totalPages": 8
  }
}
```

### 过滤

使用查询参数进行过滤：

```
GET /api/tasks?status=in_progress&assignee=user123&createdAfter=2025-01-01
```

### 部分更新（PATCH）

接受部分对象——仅更新提供的内容：

```typescript
// 仅 title 变更，其余保持不变
PATCH /api/tasks/123
{ "title": "Updated title" }
```

## TypeScript 接口模式

### 使用可辨识联合类型表示变体

```typescript
// 好的做法：每个变体都是明确的
type TaskStatus =
  | { type: 'pending' }
  | { type: 'in_progress'; assignee: string; startedAt: Date }
  | { type: 'completed'; completedAt: Date; completedBy: string }
  | { type: 'cancelled'; reason: string; cancelledAt: Date };

// 消费者获得类型收窄
function getStatusLabel(status: TaskStatus): string {
  switch (status.type) {
    case 'pending': return 'Pending';
    case 'in_progress': return `In progress (${status.assignee})`;
    case 'completed': return `Done on ${status.completedAt}`;
    case 'cancelled': return `Cancelled: ${status.reason}`;
  }
}
```

### 输入/输出分离

```typescript
// 输入：调用者提供的内容
interface CreateTaskInput {
  title: string;
  description?: string;
}

// 输出：系统返回的内容（包含服务端生成的字段）
interface Task {
  id: string;
  title: string;
  description: string | null;
  createdAt: Date;
  updatedAt: Date;
  createdBy: string;
}
```

### 使用品牌类型表示 ID

```typescript
type TaskId = string & { readonly __brand: 'TaskId' };
type UserId = string & { readonly __brand: 'UserId' };

// 防止意外将 UserId 传入期望 TaskId 的地方
function getTask(id: TaskId): Promise<Task> { ... }
```

## 常见的自我合理化

| 合理化说法 | 现实 |
|---|---|
| "我们以后再写 API 文档" | 类型就是文档。先定义它们。 |
| "我们暂时不需要分页" | 一旦有人有 100+ 条数据就会需要。从一开始就加上。 |
| "PATCH 太复杂了，直接用 PUT 吧" | PUT 每次都需要完整对象。PATCH 才是客户端真正想要的。 |
| "需要时再给 API 加版本" | 没有版本控制的破坏性变更会破坏消费者。从一开始就为扩展而设计。 |
| "没人会用那个未记录的行为" | Hyrum 定律：如果它是可观察的，就有人依赖它。将每个公共行为视为承诺。 |
| "我们可以同时维护两个版本" | 多版本会使维护成本成倍增加，并造成菱形依赖问题。优先遵循单一版本规则。 |
| "内部 API 不需要契约" | 内部消费者也是消费者。契约可以防止耦合，支持并行开发。 |

## 危险信号

- 端点根据条件返回不同的数据结构
- 不同端点间的错误格式不一致
- 验证散布在内部代码中而非边界处
- 对现有字段的破坏性变更（类型变更、移除）
- 没有分页的列表端点
- REST URL 中使用动词（`/api/createTask`、`/api/getUsers`）
- 未经验证或清理就使用第三方 API 响应

## 验证清单

设计完 API 后：

- [ ] 每个端点都有类型化的输入和输出 schema
- [ ] 错误响应遵循单一一致的格式
- [ ] 验证仅在系统边界处进行
- [ ] 列表端点支持分页
- [ ] 新字段是附加性和可选的（向后兼容）
- [ ] 命名在所有端点中遵循一致的约定
- [ ] API 文档或类型与实现一起提交
