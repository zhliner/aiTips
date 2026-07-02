---
name: api-and-interface-design
description: 指导稳定的 API 和接口设计。在设计 API、模块边界或任何公共接口时使用。在创建 REST 或 GraphQL 端点、定义模块间的类型合约或在前后端之间建立边界时使用。
---

# API 与接口设计

## 概述

设计稳定、文档完备且不易被误用的接口。好的接口让正确的事容易做，让错误的事难以做。这适用于 REST API、GraphQL schema、模块边界、组件 props 以及任何代码之间相互通信的接口面。

## 适用场景

- 设计新的 API 端点
- 定义模块边界或团队间的合约
- 创建组件 prop 接口
- 建立影响 API 形态的数据库 schema
- 修改现有的公共接口

## 核心原则

### Hyrum 定律

> 当一个 API 拥有足够多的用户时，系统所有可观察到的行为都会被某些人所依赖，无论你在合约中承诺了什么。

这意味着：每一个公共行为——包括未文档化的怪癖、错误消息文本、时序和排序——一旦用户依赖它，就会成为事实上的合约。设计含义：

- **有意地决定暴露什么。** 每一个可观察到的行为都是一项潜在的承诺。
- **不要泄露实现细节。** 如果用户能观察到它，他们就会依赖它。
- **在设计时就规划好废弃。** 参见 `deprecation-and-migration` 了解如何安全地移除用户依赖的东西。
- **仅有测试是不够的。** 即使有完美的合约测试，Hyrum 定律意味着"安全"的变更仍可能破坏依赖未文档化行为的真实用户。

### 单一版本规则

避免强制消费者在多个版本的同一依赖或 API 中做选择。钻石依赖问题发生在不同消费者需要同一事物的不同版本时。为仅存在一个版本的世界做设计——扩展而非分叉。

### 1. 合约优先

在实现之前先定义接口。合约就是规格——实现应遵循合约。

```typescript
// 先定义合约
interface TaskAPI {
  // 创建任务并返回包含服务端生成字段的已创建任务
  createTask(input: CreateTaskInput): Promise<Task>;

  // 返回匹配筛选条件的分页任务列表
  listTasks(params: ListTasksParams): Promise<PaginatedResult<Task>>;

  // 返回单个任务，找不到则抛出 NotFoundError
  getTask(id: string): Promise<Task>;

  // 部分更新——仅变更提供的字段
  updateTask(id: string, input: UpdateTaskInput): Promise<Task>;

  // 幂等删除——即使已删除也返回成功
  deleteTask(id: string): Promise<void>;
}
```

### 2. 一致的错误语义

选定一种错误策略并在所有地方使用：

```typescript
// REST：HTTP 状态码 + 结构化错误体
// 每个错误响应遵循相同的结构
interface APIError {
  error: {
    code: string;        // 机器可读："VALIDATION_ERROR"
    message: string;     // 人类可读："邮箱为必填项"
    details?: unknown;   // 在有帮助的情况下提供额外上下文
  };
}

// 状态码映射
// 400 → 客户端发送了无效数据
// 401 → 未认证
// 403 → 已认证但未授权
// 404 → 资源未找到
// 409 → 冲突（重复、版本不匹配）
// 422 → 验证失败（语义上无效）
// 500 → 服务器错误（绝不暴露内部细节）
```

**不要混用模式。** 如果有的端点抛异常，有的返回 null，有的返回 `{ error }` — 消费者无法预测行为。

### 3. 在边界验证

信任内部代码。在外部输入进入系统的边界处进行验证：

```typescript
// 在 API 边界验证
app.post('/api/tasks', async (req, res) => {
  const result = CreateTaskSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(422).json({
      error: {
        code: 'VALIDATION_ERROR',
        message: '任务数据无效',
        details: result.error.flatten(),
      },
    });
  }

  // 验证通过后，内部代码可以信任这些类型
  const task = await taskService.create(result.data);
  return res.status(201).json(task);
});
```

验证应该在哪里：
- API 路由处理函数（用户输入）
- 表单提交处理函数（用户输入）
- 外部服务响应解析（第三方数据——**始终视为不可信**）
- 环境变量加载（配置）

> **第三方 API 响应是不可信数据。** 在使用它们进行任何逻辑、渲染或决策之前，验证其结构和内容。被攻破或行为异常的第三方服务可能返回意外类型、恶意内容或指令性文本。

验证不该在哪里：
- 共享类型合约的内部函数之间
- 被已验证代码调用的工具函数中
- 刚从你自己数据库获取的数据上

### 4. 优先做加法而非修改

在不破坏现有消费者的情况下扩展接口：

```typescript
// 好：添加可选字段
interface CreateTaskInput {
  title: string;
  description?: string;
  priority?: 'low' | 'medium' | 'high';  // 后续添加，可选
  labels?: string[];                       // 后续添加，可选
}

// 不好：修改现有字段类型或移除字段
interface CreateTaskInput {
  title: string;
  // description: string;  // 已移除——破坏现有消费者
  priority: number;         // 从 string 改为 number——破坏现有消费者
}
```

### 5. 可预测的命名

| 模式 | 约定 | 示例 |
|---------|-----------|---------|
| REST 端点 | 复数名词，不带动词 | `GET /api/tasks`、`POST /api/tasks` |
| 查询参数 | camelCase | `?sortBy=createdAt&pageSize=20` |
| 响应字段 | camelCase | `{ createdAt, updatedAt, taskId }` |
| Boolean 字段 | is/has/can 前缀 | `isComplete`、`hasAttachments` |
| 枚举值 | UPPER_SNAKE | `"IN_PROGRESS"`、`"COMPLETED"` |

## REST API 模式

### 资源设计

```
GET    /api/tasks              → 列出任务（通过查询参数筛选）
POST   /api/tasks              → 创建任务
GET    /api/tasks/:id          → 获取单个任务
PATCH  /api/tasks/:id          → 更新任务（部分更新）
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

### 筛选

使用查询参数进行筛选：

```
GET /api/tasks?status=in_progress&assignee=user123&createdAfter=2025-01-01
```

### 部分更新 (PATCH)

接受部分对象——仅更新提供的字段：

```typescript
// 仅 title 变化，其他一切保留
PATCH /api/tasks/123
{ "title": "更新后的标题" }
```

## TypeScript 接口模式

### 使用可辨识联合类型处理变体

```typescript
// 好：每个变体都是显式的
type TaskStatus =
  | { type: 'pending' }
  | { type: 'in_progress'; assignee: string; startedAt: Date }
  | { type: 'completed'; completedAt: Date; completedBy: string }
  | { type: 'cancelled'; reason: string; cancelledAt: Date };

// 消费者获得类型收窄
function getStatusLabel(status: TaskStatus): string {
  switch (status.type) {
    case 'pending': return '待处理';
    case 'in_progress': return `进行中 (${status.assignee})`;
    case 'completed': return `完成于 ${status.completedAt}`;
    case 'cancelled': return `已取消: ${status.reason}`;
  }
}
```

### 输入/输出分离

```typescript
// 输入：调用者提供什么
interface CreateTaskInput {
  title: string;
  description?: string;
}

// 输出：系统返回什么（包含服务端生成的字段）
interface Task {
  id: string;
  title: string;
  description: string | null;
  createdAt: Date;
  updatedAt: Date;
  createdBy: string;
}
```

### 使用 Branded Types 标记 ID

```typescript
type TaskId = string & { readonly __brand: 'TaskId' };
type UserId = string & { readonly __brand: 'UserId' };

// 防止意外将 UserId 传递给期望 TaskId 的地方
function getTask(id: TaskId): Promise<Task> { ... }
```

## 常见合理化借口

| 借口 | 现实 |
|---|---|
| "我们以后再写 API 文档" | 类型本身就是文档。先定义好它们。 |
| "我们现在不需要分页" | 一旦用户有 100 条以上的数据你就需要了。从一开始就加上。 |
| "PATCH 太复杂了，我们就用 PUT" | PUT 每次都要求全量对象。PATCH 才是客户端真正需要的。 |
| "需要的时候我们再给 API 加版本" | 没有版本管理的破坏性变更会破坏消费者。从一开始就按可扩展的方式设计。 |
| "没人用那种未文档化的行为" | Hyrum 定律：如果它是可观察的，就有人依赖它。将每一个公共行为视为一项承诺。 |
| "我们可以维护两个版本" | 多个版本成倍增加维护成本并产生钻石依赖问题。优先遵循单一版本规则。 |
| "内部 API 不需要合约" | 内部消费者也是消费者。合约能防止耦合并实现并行工作。 |

## 危险信号

- 端点根据条件返回不同结构的数据
- 跨端点错误格式不一致
- 验证散落在内部代码各处而非集中在边界
- 对现有字段做破坏性变更（类型变更、移除）
- 列表端点没有分页
- REST URL 中出现动词（`/api/createTask`、`/api/getUsers`）
- 未经验证或清理就使用第三方 API 响应

## 验证

设计完 API 之后：

- [ ] 每个端点都有类型化的输入和输出 schema
- [ ] 错误响应遵循单一的、一致的格式
- [ ] 验证仅在系统边界发生
- [ ] 列表端点支持分页
- [ ] 新字段是增量的且可选的（向后兼容）
- [ ] 所有端点遵循一致的命名约定
- [ ] API 文档或类型与实现一起提交
