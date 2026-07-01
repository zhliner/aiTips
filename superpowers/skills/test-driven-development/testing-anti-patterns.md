# 测试反模式（Testing Anti-Patterns）

**在以下情况加载此参考：** 编写或修改测试、添加 mock、或想要向生产代码添加仅用于测试的方法时。

## 概述

测试必须验证真实行为，而不是 mock 行为。mock 是隔离的手段，而非被测试的对象。

**核心原则：** 测试代码做了什么，而非 mock 做了什么。

**遵循严格的 TDD 可防止这些反模式。**

## 铁律

```
1. 绝不测试 mock 行为
2. 绝不向生产类添加仅用于测试的方法
3. 绝不不理解依赖就使用 mock
```

## 反模式 1：测试 Mock 行为

**违规（Violation）：**
```typescript
// ❌ BAD: 测试 mock 是否存在
test('渲染侧边栏', () => {
  render(<Page />);
  expect(screen.getByTestId('sidebar-mock')).toBeInTheDocument();
});
```

**为什么这是错误的：**
- 你在验证 mock 是否工作，而非组件是否工作
- 当 mock 存在时测试通过，不存在时失败
- 对真实行为没有任何说明

**你的伙伴的纠正：** "我们在测试一个 mock 的行为吗？"

**修复（Fix）：**
```typescript
// ✅ GOOD: 测试真实组件，或者不 mock 它
test('渲染侧边栏', () => {
  render(<Page />);  // 不 mock 侧边栏
  expect(screen.getByRole('navigation')).toBeInTheDocument();
});

// 或者，如果侧边栏必须为隔离而 mock：
// 不要对 mock 做断言 — 测试 Page 在侧边栏存在时的行为
```

### 门禁函数

```
在对任何 mock 元素做断言之前：
  问："我是在测试真实组件行为还是仅测试 mock 的存在？"

  如果是在测试 mock 的存在：
    停止 — 删除断言或取消 mock 该组件

  改为测试真实行为
```

## 反模式 2：生产类中的仅测试方法

**违规（Violation）：**
```typescript
// ❌ BAD: destroy() 仅在测试中使用
class Session {
  async destroy() {  // 看起来像生产 API！
    await this._workspaceManager?.destroyWorkspace(this.id);
    // ... 清理
  }
}

// 在测试中
afterEach(() => session.destroy());
```

**为什么这是错误的：**
- 生产类被仅用于测试的代码污染
- 如果在生产环境中意外调用会很危险
- 违反 YAGNI 和关注点分离
- 混淆了对象生命周期和实体生命周期

**修复（Fix）：**
```typescript
// ✅ GOOD: 测试工具负责处理测试清理
// Session 没有 destroy() — 在生产中它是无状态的

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

### 门禁函数

```
在向生产类添加任何方法之前：
  问："这个方法是仅由测试使用的吗？"

  如果是：
    停止 — 不要添加它
    把它放在测试工具中

  问："这个类拥有这个资源的生命周期管理吗？"

  如果否：
    停止 — 这个方法不应该在这个类里
```

## 反模式 3：不经理解就使用 Mock

**违规（Violation）：**
```typescript
// ❌ BAD: Mock 破坏了测试依赖的逻辑
test('检测重复服务器', () => {
  // Mock 阻止了测试所依赖的配置写入！
  vi.mock('ToolCatalog', () => ({
    discoverAndCacheTools: vi.fn().mockResolvedValue(undefined)
  }));

  await addServer(config);
  await addServer(config);  // 应该抛出 — 但不会！
});
```

**为什么这是错误的：**
- 被 mock 的方法有测试依赖的副作用（写入配置）
- 为了"安全"而过度 mock 破坏了实际行为
- 测试因错误原因通过或神秘地失败

**修复（Fix）：**
```typescript
// ✅ GOOD: 在正确的层级 mock
test('检测重复服务器', () => {
  // mock 慢的部分，保留测试需要的行为
  vi.mock('MCPServerManager'); // 仅 mock 慢的服务器启动

  await addServer(config);  // 配置已写入
  await addServer(config);  // 检测到重复 ✓
});
```

### 门禁函数

```
在 mock 任何方法之前：
  停止 — 先不要 mock

  1. 问："真实方法有哪些副作用？"
  2. 问："这个测试是否依赖其中任何副作用？"
  3. 问："我是否完全理解这个测试需要什么？"

  如果依赖副作用：
    在更低层级 mock（实际的慢/外部操作）
    或使用保留必要行为的测试替身
    不是测试依赖的高层方法

  如果不确定测试依赖什么：
    先使用真实实现运行测试
    观察实际需要发生什么
    然后在正确的层级添加最小限度的 mock

  红色警报：
    - "我 mock 这个以保安全"
    - "这个可能很慢，最好 mock 它"
    - 不理解依赖链就使用 mock
```

## 反模式 4：不完整的 Mock

**违规（Violation）：**
```typescript
// ❌ BAD: 部分 mock — 仅包含你认为需要的字段
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' }
  // 缺失：下游代码使用的 metadata
};

// 之后：代码访问 response.metadata.requestId 时崩溃
```

**为什么这是错误的：**
- **部分 mock 隐藏了结构假设** — 你只 mock 了你已知的字段
- **下游代码可能依赖你没有包含的字段** — 静默失败
- **测试通过但集成失败** — mock 不完整，真实 API 完整
- **虚假的信心** — 测试没有证明任何关于真实行为的东西

**铁律：** mock 完整的数据结构 — 与现实一致，而不仅仅是当前测试直接使用的字段。

**修复（Fix）：**
```typescript
// ✅ GOOD: 反映真实 API 的完整性
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' },
  metadata: { requestId: 'req-789', timestamp: 1234567890 }
  // 真实 API 返回的所有字段
};
```

### 门禁函数

```
在创建 mock 响应之前：
  检查："真实 API 响应包含哪些字段？"

  行动：
    1. 从文档/示例中检查实际 API 响应
    2. 包含系统可能在下游消费的所有字段
    3. 验证 mock 完全匹配真实响应模式

  关键：
    如果你在创建 mock，你必须理解整个结构
    部分 mock 会在代码依赖省略字段时静默失败

  如果不确定：包含所有已文档化的字段
```

## 反模式 5：将集成测试视为事后补充

**违规（Violation）：**
```
✅ 实现完成
❌ 未写测试
"准备好测试了"
```

**为什么这是错误的：**
- 测试是实现的一部分，而非可选的后续步骤
- TDD 本应抓住这一点
- 没有测试就不能声称完成

**修复（Fix）：**
```
TDD 循环：
1. 编写失败的测试
2. 实现使其通过
3. 重构
4. 然后才声称完成
```

## 当 Mock 变得过于复杂时

**警告信号：**
- mock 设置比测试逻辑还长
- 为了让测试通过而 mock 所有东西
- mock 缺少真实组件的方法
- 当 mock 变化时测试崩溃

**你的伙伴会问：** "我们这里需要用 mock 吗？"

**考虑：** 使用真实组件的集成测试通常比复杂的 mock 更简单

## TDD 防止这些反模式

**为什么 TDD 有帮助：**
1. **先写测试** → 迫使你思考你实际在测试什么
2. **看着它失败** → 确认测试测试的是真实行为，而非 mock
3. **最简实现** → 不会混入仅用于测试的方法
4. **真实依赖** → 你在 mock 之前先看到测试实际需要什么

**如果你在测试 mock 行为，你违反了 TDD** — 你在没有先看到测试对真实代码失败的情况下就添加了 mock。

## 快速参考

| 反模式 | 修复 |
|--------------|-----|
| 对 mock 元素做断言 | 测试真实组件或不 mock 它 |
| 生产类中的仅测试方法 | 移至测试工具 |
| 不理解就使用 mock | 首先理解依赖，最小限度 mock |
| 不完整的 mock | 完全反映真实 API |
| 测试作为事后补充 | TDD — 测试先行 |
| 过于复杂的 mock | 考虑使用集成测试 |

## 红色警报

- 断言检查 `*-mock` 测试 ID
- 仅在测试文件中调用的方法
- mock 设置占测试 50% 以上
- 移除 mock 后测试失败
- 无法解释为什么需要 mock
- "只是为了安全"而 mock

## 底线

**mock 是隔离的工具，而非被测试的对象。**

如果 TDD 揭示你在测试 mock 行为，你就走错了。

修复：测试真实行为或质疑你为什么一开始就需要 mock。
