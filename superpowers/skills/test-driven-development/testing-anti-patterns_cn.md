# 测试反模式（Testing Anti-Patterns）

**何时加载此参考：** 编写或修改测试时，添加 mock 时，或忍不住向生产代码添加仅供测试的方法时。

## 概述

测试必须验证真实行为，而不是 mock 行为。Mock 是隔离的手段，不是被测对象。

**核心原则：** 测试代码做了什么，而不是 mock 做了什么。

**严格遵循 TDD 可以防止这些反模式。**

## 铁律

```
1. 绝不测试 mock 行为
2. 绝不向生产类添加仅供测试的方法
3. 绝不在不了解依赖关系的情况下进行 mock
```

## 反模式 1：测试 Mock 行为

**违规示例：**
```typescript
// ❌ BAD: 测试 mock 是否存在
test('渲染侧边栏', () => {
  render(<Page />);
  expect(screen.getByTestId('sidebar-mock')).toBeInTheDocument();
});
```

**为什么这是错的：**
- 你在验证 mock 是否工作，而非组件是否工作
- 测试在 mock 存在时通过，不存在时失败
- 这没有告诉你任何关于真实行为的信息

**人工伙伴的纠正：** "我们是在测试 mock 的行为吗？"

**修复方法：**
```typescript
// ✅ GOOD: 测试真实组件，或不要 mock 它
test('渲染侧边栏', () => {
  render(<Page />);  // 不要 mock sidebar
  expect(screen.getByRole('navigation')).toBeInTheDocument();
});

// 或者，如果 sidebar 确实需要为了隔离而 mock：
// 不要对 mock 元素进行断言 - 测试 Page 在有 sidebar 存在时的行为
```

### 检入函数

```
在对任何 mock 元素进行断言之前：
  问自己: "我在测试真实组件的行为还是仅测试 mock 的存在？"

  如果是在测试 mock 是否存在：
    停下 - 删除该断言或取消该组件的 mock

  改为测试真实行为
```

## 反模式 2：生产代码中的仅供测试方法

**违规示例：**
```typescript
// ❌ BAD: destroy() 只在测试中使用
class Session {
  async destroy() {  // 看起来像生产 API！
    await this._workspaceManager?.destroyWorkspace(this.id);
    // ... 清理
  }
}

// 在测试中
afterEach(() => session.destroy());
```

**为什么这是错的：**
- 生产类被仅供测试的代码污染
- 如果在生产环境中意外调用会有危险
- 违反 YAGNI 和关注点分离
- 混淆了对象生命周期和实体生命周期

**修复方法：**
```typescript
// ✅ GOOD: 测试工具处理测试清理
// Session 没有 destroy() - 它在生产环境中是无状态的

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

### 检入函数

```
在向生产类添加任何方法之前：
  问自己: "这只有测试在使用吗？"

  如果是：
    停下 - 不要添加它
    把它放到测试工具中

  问自己: "这个类是否拥有此资源的生命周期？"

  如果不是：
    停下 - 这个方法放错了类
```

## 反模式 3：不了解就 Mock

**违规示例：**
```typescript
// ❌ BAD: Mock 破坏了测试依赖的逻辑
test('检测重复服务器', () => {
  // Mock 阻止了测试依赖的配置写入！
  vi.mock('ToolCatalog', () => ({
    discoverAndCacheTools: vi.fn().mockResolvedValue(undefined)
  }));

  await addServer(config);
  await addServer(config);  // 应该抛出异常 - 但不会！
});
```

**为什么这是错的：**
- Mock 的方法有测试依赖的副作用（写入配置）
- 为了"安全"而过度的 mock 破坏了实际行为
- 测试因为错误的原因通过或神秘地失败

**修复方法：**
```typescript
// ✅ GOOD: 在正确的层级进行 mock
test('检测重复服务器', () => {
  // Mock 慢的部分，保留测试需要的行为
  vi.mock('MCPServerManager'); // 仅 mock 慢的服务器启动

  await addServer(config);  // 配置已写入
  await addServer(config);  // 检测到重复 ✓
});
```

### 检入函数

```
在对任何方法进行 mock 之前：
  停下 - 先不要 mock

  1. 问自己: "真实方法有哪些副作用？"
  2. 问自己: "此测试是否依赖这些副作用中的任何一个？"
  3. 问自己: "我是否完全理解此测试需要什么？"

  如果依赖副作用：
    在更低层级进行 mock（实际的慢/外部操作）
    或使用保留必要行为的测试替身
    而非测试依赖的高层方法

  如果不确定测试依赖什么：
    先用真实实现运行测试
    观察实际上需要发生什么
    然后在正确层级添加最小化的 mock

  危险信号：
    - "为安全起见，我对这个做 mock"
    - "这可能很慢，最好 mock 掉它"
    - 在不了解依赖链的情况下进行 mock
```

## 反模式 4：不完整的 Mock

**违规示例：**
```typescript
// ❌ BAD: 部分 mock - 只提供了你认为需要的字段
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' }
  // 缺失：下游代码使用的 metadata
};

// 之后：当代码访问 response.metadata.requestId 时出错
```

**为什么这是错的：**
- **部分 mock 隐藏了结构假设** - 你只 mock 了你已知的字段
- **下游代码可能依赖你没有包含的字段** - 静默失败
- **测试通过但集成失败** - mock 不完整，真实 API 完整
- **虚假信心** - 测试没有证明任何关于真实行为的东西

**铁律：** Mock 真实存在的完整数据结构，而不仅仅是你当前测试使用的字段。

**修复方法：**
```typescript
// ✅ GOOD: 镜像真实 API 的完整性
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' },
  metadata: { requestId: 'req-789', timestamp: 1234567890 }
  // 真实 API 返回的全部字段
};
```

### 检入函数

```
在创建 mock 响应之前：
  检查: "真实 API 响应包含哪些字段？"

  行动：
    1. 从文档/示例中检查实际 API 响应
    2. 包含系统下游可能消费的所有字段
    3. 验证 mock 与真实响应结构完全匹配

  关键：
    如果你在创建 mock，你必须了解整个结构
    当下游代码依赖被遗漏的字段时，部分 mock 会静默失败

  如果不确定：包含所有文档记录的字段
```

## 反模式 5：事后考虑的集成测试

**违规示例：**
```
✅ 实现完成
❌ 没有编写测试
"准备测试"
```

**为什么这是错的：**
- 测试是实现的一部分，不是可选的后续
- TDD 本会捕获这一点
- 没有测试就不能声称完成

**修复方法：**
```
TDD 循环：
1. 编写失败测试
2. 实现使其通过
3. 重构
4. 然后声称完成
```

## 当 Mock 变得过于复杂时

**警告信号：**
- Mock 设置比测试逻辑更长
- 为了使测试通过而 mock 一切
- Mock 缺少真实组件具有的方法
- 测试在 mock 改变时失败

**人工伙伴的问题：** "这里我们是否真的需要使用 mock？"

**考虑：** 使用真实组件的集成测试通常比复杂 mock 更简单

## TDD 如何防止这些反模式

**为什么 TDD 有帮助：**
1. **先写测试** → 迫使你思考你实际上在测试什么
2. **看着它失败** → 确认测试检验的是真实行为，而非 mock
3. **最小实现** → 不会混入仅供测试的方法
4. **真实依赖** → 你在 mock 之前看到了测试实际需要什么

**如果你在测试 mock 行为，你已经违反了 TDD** - 你在没有先看着测试对真实代码失败的情况下就添加了 mock。

## 快速参考

| 反模式 | 修复 |
|--------------|-----|
| 对 mock 元素进行断言 | 测试真实组件或取消 mock |
| 生产代码中的仅供测试方法 | 移至测试工具中 |
| 不了解就 mock | 先理解依赖，最小化 mock |
| 不完整的 mock | 完全镜像真实 API |
| 事后考虑的测试 | TDD - 测试先行 |
| 过于复杂的 mock | 考虑集成测试 |

## 红线

- 断言检查 `*-mock` 测试 ID
- 方法仅在测试文件中被调用
- Mock 设置占测试代码的 50% 以上
- 移除 mock 后测试失败
- 无法解释为何需要 mock
- "为了安全起见"而 mock

## 底线

**Mock 是隔离的工具，不是被测对象。**

如果 TDD 揭示你正在测试 mock 行为，你已经走偏了。

修复：测试真实行为，或质疑你究竟为什么要用 mock。
