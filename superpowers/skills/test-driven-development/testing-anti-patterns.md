# Testing Anti-Patterns（测试反模式）

**加载此参考时机：** 编写或更改测试、添加 mock、或想向生产代码添加仅测试方法时。

## Overview（概述）

测试必须验证真实行为，不是 mock 行为。Mock 是隔离的手段，不是被测试的东西。

**核心原则：** 测试代码做什么，不是 mock 做什么。

**遵循严格的 TDD 防止这些反模式。**

## The Iron Laws（铁律）

```
1. 永远不要测试 mock 行为
2. 永远不要向生产类添加仅测试方法
3. 永远不要在不理解依赖的情况下 mock
```

## Anti-Pattern 1: Testing Mock Behavior（反模式 1：测试 Mock 行为）

**违规：**
```typescript
// ❌ 坏：测试 mock 存在
test('renders sidebar', () => {
  render(<Page />);
  expect(screen.getByTestId('sidebar-mock')).toBeInTheDocument();
});
```

**为什么这是错的：**
- 你在验证 mock 工作，不是组件工作
- 测试在 mock 存在时通过，不存在时失败
- 对真实行为一无所知

**your human partner's correction：** "Are we testing the behavior of a mock?"

**修复：**
```typescript
// ✅ 好：测试真实组件或不要 mock 它
test('renders sidebar', () => {
  render(<Page />);  // 不要 mock sidebar
  expect(screen.getByRole('navigation')).toBeInTheDocument();
});

// 或者如果 sidebar 必须被 mock 以隔离：
// 不要断言 mock - 测试 Page 在 sidebar 存在时的行为
```

### Gate Function（门控函数）

```
在对任何 mock 元素断言之前：
  问："我在测试真实组件行为还是只是 mock 存在？"

  如果测试 mock 存在：
    停止 - 删除断言或取消 mock 组件

  改为测试真实行为
```

## Anti-Pattern 2: Test-Only Methods in Production（反模式 2：生产中的仅测试方法）

**违规：**
```typescript
// ❌ 坏：destroy() 仅在测试中使用
class Session {
  async destroy() {  // 看起来像生产 API！
    await this._workspaceManager?.destroyWorkspace(this.id);
    // ... cleanup
  }
}

// 在测试中
afterEach(() => session.destroy());
```

**为什么这是错的：**
- 生产类被仅测试代码污染
- 如果在生产中意外调用则危险
- 违反 YAGNI 和关注点分离
- 混淆对象生命周期与实体生命周期

**修复：**
```typescript
// ✅ 好：测试实用程序处理测试清理
// Session 没有 destroy() - 在生产中它是无状态的

// 在 test-utils/ 中
export async function cleanupSession(session: Session) {
  const workspace = session.getWorkspaceInfo();
  if (workspace) {
    await workspaceManager.destroyWorkspace(workspace.id);
  }
}

// 在测试中
afterEach(() => cleanupSession(session));
```

### Gate Function（门控函数）

```
在向生产类添加任何方法之前：
  问："这只被测试使用吗？"

  如果是：
    停止 - 不要添加它
    改为放在测试实用程序中

  问："这个类拥有这个资源的生命周期吗？"

  如果不是：
    停止 - 此方法的错误类
```

## Anti-Pattern 3: Mocking Without Understanding（反模式 3：不理解地 Mock）

**违规：**
```typescript
// ❌ 坏：Mock 破坏测试逻辑
test('detects duplicate server', () => {
  // Mock 阻止测试依赖的配置写入！
  vi.mock('ToolCatalog', () => ({
    discoverAndCacheTools: vi.fn().mockResolvedValue(undefined)
  }));

  await addServer(config);
  await addServer(config);  // 应该抛出 - 但不会！
});
```

**为什么这是错的：**
- Mock 方法有测试依赖的副作用（写入配置）
- 过度 mock"为了安全"破坏实际行为
- 测试因错误原因通过或神秘失败

**修复：**
```typescript
// ✅ 好：在正确的层级 Mock
test('detects duplicate server', () => {
  // Mock 慢的部分，保留测试需要的行为
  vi.mock('MCPServerManager'); // 只 mock 慢的服务器启动

  await addServer(config);  // 配置已写入
  await addServer(config);  // 检测到重复 ✓
});
```

### Gate Function（门控函数）

```
在 mock 任何方法之前：
  停止 - 还不要 mock

  1. 问："真实方法有什么副作用？"
  2. 问："这个测试依赖那些副作用吗？"
  3. 问："我完全理解这个测试需要什么吗？"

  如果依赖副作用：
    在更低层级 mock（实际慢/外部操作）
    或保留必要行为的测试替身
    不是测试依赖的高层级方法

  如果不确定测试依赖什么：
    首先用真实实现运行测试
    观察实际需要发生什么
    然后在正确层级添加最小 mock

  红旗：
    - "I'll mock this to be safe"
    - "This might be slow, better mock it"
    - 在不理解依赖链的情况下 mock
```

## Anti-Pattern 4: Incomplete Mocks（反模式 4：不完整的 Mock）

**违规：**
```typescript
// ❌ 坏：部分 mock - 只有你认为需要的字段
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' }
  // 缺失：下游代码使用的元数据
};

// 之后：当代码访问 response.metadata.requestId 时中断
```

**为什么这是错的：**
- **部分 mock 隐藏结构假设** - 你只 mock 了你知道的字段
- **下游代码可能依赖你没有包含的字段** - 静默失败
- **测试通过但集成失败** - Mock 不完整，真实 API 完整
- **虚假的信心** - 测试对真实行为一无所知

**铁律：** Mock 完整的数据结构，就像它在现实中存在的那样，不只是你直接测试使用的字段。

**修复：**
```typescript
// ✅ 好：镜像真实 API 完整性
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' },
  metadata: { requestId: 'req-789', timestamp: 1234567890 }
  // 真实 API 返回的所有字段
};
```

### Gate Function（门控函数）

```
在创建 mock 响应之前：
  检查："真实 API 响应包含什么字段？"

  行动：
    1. 检查文档/示例中的实际 API 响应
    2. 包含系统下游可能消费的所有字段
    3. 验证 mock 完全匹配真实响应模式

  关键：
    如果你在创建 mock，你必须理解完整结构
    部分 mock 在代码依赖省略字段时静默失败

  如果不确定：包含所有记录的字段
```

## Anti-Pattern 5: Integration Tests as Afterthought（反模式 5：集成测试作为事后想法）

**违规：**
```
✅ Implementation complete
❌ No tests written
"Ready for testing"
```

**为什么这是错的：**
- 测试是实现的一部分，不是可选的后续
- TDD 会捕获这个
- 没有测试就不能声称完成

**修复：**
```
TDD 循环：
1. 编写失败测试
2. 实现以通过
3. 重构
4. 然后声称完成
```

## When Mocks Become Too Complex（当 Mock 变得过于复杂时）

**警告信号：**
- Mock 设置比测试逻辑长
- Mock 所有东西以使测试通过
- Mock 缺少真实组件有的方法
- 测试在 mock 更改时中断

**your human partner's question：** "Do we need to be using a mock here?"

**考虑：** 带真实组件的集成测试通常比复杂 mock 更简单

## TDD Prevents These Anti-Patterns（TDD 防止这些反模式）

**为什么 TDD 有帮助：**
1. **先写测试** → 迫使你思考你实际在测试什么
2. **观察它失败** → 确认测试测试真实行为，不是 mock
3. **最小实现** → 仅测试方法不会悄悄进入
4. **真实依赖** → 你在 mock 之前看到测试实际需要什么

**如果你在测试 mock 行为，你违反了 TDD** - 你在没有观察测试对真实代码失败的情况下添加了 mock。

## Quick Reference（快速参考）

| 反模式 | 修复 |
|--------------|-----|
| 断言 mock 元素 | 测试真实组件或取消 mock |
| 生产中的仅测试方法 | 移到测试实用程序 |
| 不理解地 mock | 先理解依赖，最小 mock |
| 不完整的 mock | 完全镜像真实 API |
| 测试作为事后想法 | TDD - 测试优先 |
| 过于复杂的 mock | 考虑集成测试 |

## Red Flags（红旗）

- 断言检查 `*-mock` 测试 ID
- 方法仅在测试文件中调用
- Mock 设置 > 测试的 50%
- 测试在你移除 mock 时失败
- 无法解释为什么需要 mock
- "just to be safe"地 mock

## The Bottom Line（底线）

**Mock 是隔离的工具，不是测试的东西。**

如果 TDD 揭示你在测试 mock 行为，你走错了。

修复：测试真实行为或质疑你为什么要 mock。
