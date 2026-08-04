# 测试反模式

**在以下情况加载此参考：** 编写或修改测试、添加 mock，或想在生产代码中添加仅用于测试的方法时。

## 概述

测试必须验证真实行为，而非 mock 行为。Mock 是用于隔离的手段，而非被测试的对象。

**核心原则：** 测试代码做了什么，而非 mock 做了什么。

**遵循严格的 TDD 可以防止这些反模式。**

## 铁律

```
1. 绝不测试 mock 行为
2. 绝不在生产类中添加仅用于测试的方法
3. 绝不在不理解依赖的情况下使用 mock
```

## 反模式 1：测试 Mock 行为

**违规示例：**
```typescript
// ❌ BAD: Testing that the mock exists
test('renders sidebar', () => {
  render(<Page />);
  expect(screen.getByTestId('sidebar-mock')).toBeInTheDocument();
});
```

**错误原因：**
- 你在验证 mock 是否工作，而非组件是否工作
- 测试在 mock 存在时通过，不存在时失败
- 对真实行为没有任何说明

**your human partner's correction：** "Are we testing the behavior of a mock?"

**修正方法：**
```typescript
// ✅ GOOD: Test real component or don't mock it
test('renders sidebar', () => {
  render(<Page />);  // Don't mock sidebar
  expect(screen.getByRole('navigation')).toBeInTheDocument();
});

// OR if sidebar must be mocked for isolation:
// Don't assert on the mock - test Page's behavior with sidebar present
```

### 门控函数

```
在对任何 mock 元素进行断言之前：
  问自己："我在测试真实组件行为，还是仅仅测试 mock 的存在？"

  如果在测试 mock 的存在：
    停止 - 删除该断言或取消对组件的 mock

  改为测试真实行为
```

## 反模式 2：在生产代码中添加仅用于测试的方法

**违规示例：**
```typescript
// ❌ BAD: destroy() only used in tests
class Session {
  async destroy() {  // Looks like production API!
    await this._workspaceManager?.destroyWorkspace(this.id);
    // ... cleanup
  }
}

// In tests
afterEach(() => session.destroy());
```

**错误原因：**
- 生产类被仅用于测试的代码污染
- 如果在生产环境中被意外调用，存在风险
- 违反 YAGNI 和关注点分离原则
- 混淆了对象生命周期与实体生命周期

**修正方法：**
```typescript
// ✅ GOOD: Test utilities handle test cleanup
// Session has no destroy() - it's stateless in production

// In test-utils/
export async function cleanupSession(session: Session) {
  const workspace = session.getWorkspaceInfo();
  if (workspace) {
    await workspaceManager.destroyWorkspace(workspace.id);
  }
}

// In tests
afterEach(() => cleanupSession(session));
```

### 门控函数

```
在向生产类添加任何方法之前：
  问自己："这个方法是否仅被测试使用？"

  如果是：
    停止 - 不要添加
    将其放入测试工具中

  问自己："这个类是否拥有该资源的生命周期？"

  如果不是：
    停止 - 这个方法放错了类
```

## 反模式 3：在不理解的情况下使用 Mock

**违规示例：**
```typescript
// ❌ BAD: Mock breaks test logic
test('detects duplicate server', () => {
  // Mock prevents config write that test depends on!
  vi.mock('ToolCatalog', () => ({
    discoverAndCacheTools: vi.fn().mockResolvedValue(undefined)
  }));

  await addServer(config);
  await addServer(config);  // Should throw - but won't!
});
```

**错误原因：**
- 被 mock 的方法有测试所依赖的副作用（写入配置）
- 为了"安全"而过度 mock 破坏了实际行为
- 测试因错误原因通过，或莫名其妙地失败

**修正方法：**
```typescript
// ✅ GOOD: Mock at correct level
test('detects duplicate server', () => {
  // Mock the slow part, preserve behavior test needs
  vi.mock('MCPServerManager'); // Just mock slow server startup

  await addServer(config);  // Config written
  await addServer(config);  // Duplicate detected ✓
});
```

### 门控函数

```
在对任何方法进行 mock 之前：
  停止 - 先不要 mock

  1. 问自己："真实方法有哪些副作用？"
  2. 问自己："这个测试是否依赖其中某些副作用？"
  3. 问自己："我是否完全理解这个测试需要什么？"

  如果依赖副作用：
    在更低层级进行 mock（实际缓慢/外部操作）
    或使用保留必要行为的测试替身
    而非测试所依赖的高层方法

  如果不确定测试依赖什么：
    先用真实实现运行测试
    观察实际需要发生什么
    然后在正确层级添加最小化的 mock

  危险信号：
    - "我 mock 这个是为了安全"
    - "这个可能很慢，最好 mock 掉"
    - 在不理解依赖链的情况下使用 mock
```

## 反模式 4：不完整的 Mock

**违规示例：**
```typescript
// ❌ BAD: Partial mock - only fields you think you need
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' }
  // Missing: metadata that downstream code uses
};

// Later: breaks when code accesses response.metadata.requestId
```

**错误原因：**
- **部分 mock 隐藏了结构假设** - 你只 mock 了你知道的字段
- **下游代码可能依赖你未包含的字段** - 静默失败
- **测试通过但集成失败** - Mock 不完整，真实 API 完整
- **虚假的信心** - 测试对真实行为没有任何证明

**铁律：** Mock 完整的数据结构，如同它在现实中的样子，而非仅 mock 你当前测试使用的字段。

**修正方法：**
```typescript
// ✅ GOOD: Mirror real API completeness
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' },
  metadata: { requestId: 'req-789', timestamp: 1234567890 }
  // All fields real API returns
};
```

### 门控函数

```
在创建 mock 响应之前：
  检查："真实 API 响应包含哪些字段？"

  操作：
    1. 查看文档/示例中的实际 API 响应
    2. 包含系统下游可能消费的所有字段
    3. 验证 mock 完全匹配真实响应的 schema

  关键：
    如果你在创建 mock，你必须理解完整结构
    部分 mock 在代码依赖被省略字段时会静默失败

  如果不确定：包含所有已记录的字段
```

## 反模式 5：将集成测试当作事后补充

**违规示例：**
```
✅ 实现完成
❌ 未编写测试
"准备好进行测试了"
```

**错误原因：**
- 测试是实现的一部分，而非可选的后续步骤
- TDD 本可以避免这种情况
- 没有测试就不能声称完成

**修正方法：**
```
TDD 循环：
1. 编写失败的测试
2. 实现使其通过
3. 重构
4. 然后才能声称完成
```

## 当 Mock 变得过于复杂时

**警告信号：**
- Mock 设置比测试逻辑更长
- 为了让测试通过而 mock 一切
- Mock 缺少真实组件具有的方法
- 修改 mock 后测试就崩溃

**your human partner's question：** "Do we need to be using a mock here?"

**考虑：** 使用真实组件的集成测试通常比复杂的 mock 更简单

## TDD 如何防止这些反模式

**TDD 为何有效：**
1. **先写测试** → 迫使你思考实际要测试什么
2. **观察失败** → 确认测试验证的是真实行为，而非 mock
3. **最小化实现** → 不会有仅用于测试的方法悄悄混入
4. **真实依赖** → 在 mock 之前你能看到测试实际需要什么

**如果你在测试 mock 行为，说明你违反了 TDD** - 你在没有观察测试对真实代码失败的情况下就添加了 mock。

## 快速参考

| 反模式 | 修正方法 |
|--------|----------|
| 对 mock 元素进行断言 | 测试真实组件或取消 mock |
| 在生产代码中添加仅用于测试的方法 | 移至测试工具 |
| 在不理解的情况下使用 mock | 先理解依赖，最小化 mock |
| 不完整的 mock | 完整镜像真实 API |
| 将测试当作事后补充 | TDD - 测试先行 |
| 过于复杂的 mock | 考虑集成测试 |

## 危险信号

- 断言检查 `*-mock` 测试 ID
- 方法仅在测试文件中被调用
- Mock 设置占测试的 >50%
- 移除 mock 后测试失败
- 无法解释为什么需要 mock
- "为了安全"而 mock

## 底线

**Mock 是用于隔离的工具，而非被测试的对象。**

如果 TDD 揭示你在测试 mock 行为，说明你走错了方向。

修正：测试真实行为，或质疑为什么要 mock。
