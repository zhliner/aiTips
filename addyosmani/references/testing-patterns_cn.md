# Testing Patterns Reference（测试模式参考）

全栈常用测试模式快速参考。配合 `test-driven-development` skill 使用。

## 目录

- [Test Structure（测试结构——Arrange-Act-Assert）](#test-structure-测试结构arrange-act-assert)
- [Test Naming Conventions（测试命名规范）](#test-naming-conventions-测试命名规范)
- [Common Assertions（常用断言）](#common-assertions-常用断言)
- [Mocking Patterns（Mock 模式）](#mocking-patterns-mock-模式)
- [React/Component Testing（React/组件测试）](#reactcomponent-testing-react组件测试)
- [API / Integration Testing（API / 集成测试）](#api--integration-testing-api--集成测试)
- [E2E Testing（E2E 测试——Playwright）](#e2e-testing-e2e-测试playwright)
- [Test Anti-Patterns（测试反模式）](#test-anti-patterns-测试反模式)

## Test Structure（测试结构——Arrange-Act-Assert）

```typescript
it('describes expected behavior', () => {
  // Arrange：设置测试数据和前置条件
  const input = { title: 'Test Task', priority: 'high' };

  // Act：执行被测试的操作
  const result = createTask(input);

  // Assert：验证结果
  expect(result.title).toBe('Test Task');
  expect(result.priority).toBe('high');
  expect(result.status).toBe('pending');
});
```

## Test Naming Conventions（测试命名规范）

```typescript
// 模式：[单元] [期望行为] [条件]
describe('TaskService.createTask', () => {
  it('creates a task with default pending status', () => {});
  it('throws ValidationError when title is empty', () => {});
  it('trims whitespace from title', () => {});
  it('generates a unique ID for each task', () => {});
});
```

## Common Assertions（常用断言）

```typescript
// 相等性
expect(result).toBe(expected);           // 严格相等（===）
expect(result).toEqual(expected);        // 深度相等（对象/数组）
expect(result).toStrictEqual(expected);  // 深度相等 + 类型匹配

// 真值
expect(result).toBeTruthy();
expect(result).toBeFalsy();
expect(result).toBeNull();
expect(result).toBeDefined();
expect(result).toBeUndefined();

// 数值
expect(result).toBeGreaterThan(5);
expect(result).toBeLessThanOrEqual(10);
expect(result).toBeCloseTo(0.3, 5);      // 浮点数

// 字符串
expect(result).toMatch(/pattern/);
expect(result).toContain('substring');

// 数组 / 对象
expect(array).toContain(item);
expect(array).toHaveLength(3);
expect(object).toHaveProperty('key', 'value');

// 错误
expect(() => fn()).toThrow();
expect(() => fn()).toThrow(ValidationError);
expect(() => fn()).toThrow('specific message');

// 异步
await expect(asyncFn()).resolves.toBe(value);
await expect(asyncFn()).rejects.toThrow(Error);
```

## Mocking Patterns（Mock 模式）

### Mock Functions（Mock 函数）

```typescript
const mockFn = jest.fn();
mockFn.mockReturnValue(42);
mockFn.mockResolvedValue({ data: 'test' });
mockFn.mockImplementation((x) => x * 2);

expect(mockFn).toHaveBeenCalled();
expect(mockFn).toHaveBeenCalledWith('arg1', 'arg2');
expect(mockFn).toHaveBeenCalledTimes(3);
```

### Mock Modules（Mock 模块）

```typescript
// Mock 整个模块
jest.mock('./database', () => ({
  query: jest.fn().mockResolvedValue([{ id: 1, title: 'Test' }]),
}));

// Mock 特定导出
jest.mock('./utils', () => ({
  ...jest.requireActual('./utils'),
  generateId: jest.fn().mockReturnValue('test-id'),
}));
```

### Mock at Boundaries Only（仅在边界处 Mock）

```
Mock 以下：                      不要 Mock 以下：
├── 数据库调用                    ├── 内部工具函数
├── HTTP 请求                     ├── 业务逻辑
├── 文件系统操作                  ├── 数据转换
├── 外部 API 调用                 ├── 验证函数
└── 时间/日期（必要时）           └── 纯函数
```

## React/Component Testing（React/组件测试）

```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';

describe('TaskForm', () => {
  it('submits the form with entered data', async () => {
    const onSubmit = jest.fn();
    render(<TaskForm onSubmit={onSubmit} />);

    // 通过无障碍 role/label 查找元素（而非 test ID）
    await screen.findByRole('textbox', { name: /title/i });
    fireEvent.change(screen.getByRole('textbox', { name: /title/i }), {
      target: { value: 'New Task' },
    });
    fireEvent.click(screen.getByRole('button', { name: /create/i }));

    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalledWith({ title: 'New Task' });
    });
  });

  it('shows validation error for empty title', async () => {
    render(<TaskForm onSubmit={jest.fn()} />);

    fireEvent.click(screen.getByRole('button', { name: /create/i }));

    expect(await screen.findByText(/title is required/i)).toBeInTheDocument();
  });
});
```

## API / Integration Testing（API / 集成测试）

```typescript
import request from 'supertest';
import { app } from '../src/app';

describe('POST /api/tasks', () => {
  it('creates a task and returns 201', async () => {
    const response = await request(app)
      .post('/api/tasks')
      .send({ title: 'Test Task' })
      .set('Authorization', `Bearer ${testToken}`)
      .expect(201);

    expect(response.body).toMatchObject({
      id: expect.any(String),
      title: 'Test Task',
      status: 'pending',
    });
  });

  it('returns 422 for invalid input', async () => {
    const response = await request(app)
      .post('/api/tasks')
      .send({ title: '' })
      .set('Authorization', `Bearer ${testToken}`)
      .expect(422);

    expect(response.body.error.code).toBe('VALIDATION_ERROR');
  });

  it('returns 401 without authentication', async () => {
    await request(app)
      .post('/api/tasks')
      .send({ title: 'Test' })
      .expect(401);
  });
});
```

## E2E Testing（E2E 测试——Playwright）

```typescript
import { test, expect } from '@playwright/test';

test('user can create and complete a task', async ({ page }) => {
  // 导航并登录
  await page.goto('/');
  await page.fill('[name="email"]', 'test@example.com');
  await page.fill('[name="password"]', 'testpass123');
  await page.click('button:has-text("Log in")');

  // 创建任务
  await page.click('button:has-text("New Task")');
  await page.fill('[name="title"]', 'Buy groceries');
  await page.click('button:has-text("Create")');

  // 验证任务出现
  await expect(page.locator('text=Buy groceries')).toBeVisible();

  // 完成任务
  await page.click('[aria-label="Complete Buy groceries"]');
  await expect(page.locator('text=Buy groceries')).toHaveCSS(
    'text-decoration-line', 'line-through'
  );
});
```

## Test Anti-Patterns（测试反模式）

| 反模式 | 问题 | 更好的做法 |
|---|---|---|
| 测试实现细节 | 重构时会失败 | 测试输入/输出 |
| 所有内容都做 snapshot | 没人审查 snapshot diff | 断言具体值 |
| 共享可变状态 | 测试之间互相污染 | 每个测试独立 setup/teardown |
| 测试第三方代码 | 浪费时间，不是你的 bug | 在边界处 Mock |
| 跳过测试以通过 CI | 掩盖真正的 bug | 修复或删除测试 |
| 永久使用 `test.skip` | 死代码 | 删除或修复它 |
| 过于宽泛的断言 | 无法捕获回归问题 | 断言要具体 |
| 无异步错误处理 | 错误被吞没，假通过 | 始终 `await` 异步测试 |
