---
name: code-simplification
description: 为了清晰性而简化代码。在不改变行为的情况下重构代码以提高清晰度时使用。当代码能运行但阅读、维护或扩展起来比应有的更难时使用。在审查积累了不必要复杂性的代码时使用。
---

# 代码简化

> 灵感来自 [Claude Code Simplifier 插件](https://github.com/anthropics/claude-plugins-official/blob/main/plugins/code-simplifier/agents/code-simplifier.md)。此处改编为适用于任何 AI 编码 agent 的模型无关、流程驱动的 skill。

## 概述

通过降低复杂性同时精确保持行为来简化代码。目标不是更少的行数——是更容易阅读、理解、修改和调试的代码。每个简化都必须通过一个简单的测试："新团队成员理解这个会比理解原版更快吗？"

## 何时使用

- 功能已完成、测试已通过，但实现感觉比需要的更重时
- 代码审查中可读性或复杂性被标记时
- 遇到深度嵌套逻辑、过长函数或不清 晰的命名时
- 重构在时间压力下编写的代码时
- 整合分散在多个文件中的相关逻辑时
- 合并引入了重复或不一致的变更后

**何时不使用：**

- 代码已经干净可读——不要为了简化而简化
- 你还不知道代码做什么——先理解，再简化
- 代码是性能关键的，"更简单"的版本会明显更慢
- 你即将完全重写该模块——简化即将抛弃的代码是浪费精力

## 五大原则

### 1. 精确保持行为

不要改变代码做什么——只改变它如何表达。所有输入、输出、副作用、错误行为和边缘情况必须保持相同。如果你不确定某个简化是否保持行为，就不要做。

```
每次变更前问自己：
→ 这对每个输入产生相同的输出吗？
→ 这保持相同的错误行为吗？
→ 这保持相同的副作用和顺序吗？
→ 所有现有测试是否在不修改的情况下通过？
```

### 2. 遵循项目约定

简化意味着使代码更符合代码库，而非强加外部偏好。简化前：

```
1. 阅读 CLAUDE.md / 项目约定
2. 研究相邻代码如何处理类似模式
3. 在以下方面匹配项目风格：
   - 导入顺序和模块系统
   - 函数声明风格
   - 命名约定
   - 错误处理模式
   - 类型注解深度
```

打破项目一致性的简化不是简化——是折腾。

### 3. 优先清晰度而非巧妙

当紧凑版本需要心理停顿来解析时，显式代码优于紧凑代码。

```typescript
// UNCLEAR: 密集的三元链
const label = isNew ? 'New' : isUpdated ? 'Updated' : isArchived ? 'Archived' : 'Active';

// CLEAR: 可读的映射
function getStatusLabel(item: Item): string {
  if (item.isNew) return 'New';
  if (item.isUpdated) return 'Updated';
  if (item.isArchived) return 'Archived';
  return 'Active';
}
```

```typescript
// UNCLEAR: 带内联逻辑的串联 reduce
const result = items.reduce((acc, item) => ({
  ...acc,
  [item.id]: { ...acc[item.id], count: (acc[item.id]?.count ?? 0) + 1 }
}), {});

// CLEAR: 命名的中间步骤
const countById = new Map<string, number>();
for (const item of items) {
  countById.set(item.id, (countById.get(item.id) ?? 0) + 1);
}
```

### 4. 保持平衡

简化有一个失败模式：过度简化。警惕这些陷阱：

- **过于激进地内联**——删除赋予概念名称的辅助函数会使调用点更难读
- **合并不相关的逻辑**——两个简单函数合并成一个复杂函数并非更简单
- **删除"不必要"的抽象**——有些抽象存在是为了可扩展性或可测试性，而非复杂性
- **优化行数**——更少的行数不是目标；更容易理解才是

### 5. 限定在已变更范围内

默认只简化最近修改的代码。避免对无关代码的顺手重构，除非明确被要求扩大范围。无范围的简化会在 diff 中产生噪音，并有引发意外回归的风险。

## 简化流程

### 第 1 步：在动手前理解（切斯特顿之篱）

在更改或删除任何东西之前，理解它为什么存在。这就是切斯特顿之篱：如果你看到路上有篱笆而不理解它为什么在那里，不要拆掉它。先理解原因，再决定原因是否仍然成立。

```
简化前，回答：
- 这段代码的职责是什么？
- 什么调用它？它调用什么？
- 边缘情况和错误路径是什么？
- 有没有定义预期行为的测试？
- 为什么它可能是这样写的？（性能？平台约束？历史原因？）
- 检查 git blame：这段代码的原始上下文是什么？
```

如果你不能回答这些，你还没准备好简化。先读取更多上下文。

### 第 2 步：识别简化机会

扫描这些模式——每个模式都是一个具体信号，而非模糊的代码坏味：

**结构复杂性：**

| 模式 | 信号 | 简化 |
|---------|--------|----------------|
| 深度嵌套（3+ 层） | 难以跟踪控制流 | 将条件提取为 guard clause 或辅助函数 |
| 长函数（50+ 行） | 多重职责 | 拆分为有描述性名称的专注函数 |
| 嵌套三元表达式 | 需要心理堆栈来解析 | 替换为 if/else 链、switch 或查找对象 |
| 布尔参数标志 | `doThing(true, false, true)` | 替换为选项对象或分离的函数 |
| 重复的条件判断 | 多处出现相同的 `if` 检查 | 提取到命名良好的判定函数 |

**命名和可读性：**

| 模式 | 信号 | 简化 |
|---------|--------|----------------|
| 泛型名称 | `data`、`result`、`temp`、`val`、`item` | 重命名以描述内容：`userProfile`、`validationErrors` |
| 缩写名称 | `usr`、`cfg`、`btn`、`evt` | 使用完整单词，除非缩写在领域内是通用的（`id`、`url`、`api`） |
| 误导性名称 | 名为 `get` 但也会修改状态的函数 | 重命名以反映实际行为 |
| 解释"是什么"的注释 | `count++` 上面的 `// 计数器递增` | 删除注释——代码已经足够清晰 |
| 解释"为什么"的注释 | `// 重试因为 API 在高负载下不稳定` | 保留这些——它们承载了代码无法表达 的意图 |

**冗余：**

| 模式 | 信号 | 简化 |
|---------|--------|----------------|
| 重复逻辑 | 多处出现相同的 5+ 行代码 | 提取为共享函数 |
| 死代码 | 不可到达的分支、未使用的变量、被注释掉的块 | 删除（在确认它确实是死的之后） |
| 不必要的抽象 | 不带附加值的包装器 | 内联包装器，直接调用底层函数 |
| 过度工程化的模式 | 工厂的工厂、只有单一策略的策略模式 | 替换为简单的直接方法 |
| 冗余的类型断言 | 转换到已经推断出的类型 | 删除断言 |

### 第 3 步：增量应用变更

一次做一个简化。每次变更后运行测试。**将重构变更与功能或 bug 修复变更分开提交。** 一个既重构又添加功能的 PR 是两个 PR——拆分它们。

```
对于每个简化：
1. 进行变更
2. 运行测试套件
3. 如果测试通过 → 提交（或继续下一个简化）
4. 如果测试失败 → 回滚并重新考虑
```

避免将多个简化批处理成单个未经测试的变更。如果出错了，你需要知道是哪个简化导致的。

**五百行规则：** 如果一次重构将触及超过 500 行，投入自动化手段（codemod、sed 脚本、AST 转换）而非手动进行变更。这种规模的手动编辑容易出错，且审查起来令人疲惫。

### 第 4 步：验证结果

所有简化完成后，退后一步评估整体：

```
比较前后：
- 简化后的版本是否真正更容易理解？
- 你是否引入了与代码库不一致的新模式？
- Diff 是否干净且可审查？
- 队友会批准这个变更吗？
```

如果"简化"版本更难理解或审查，回滚。并非每次简化尝试都会成功。

## 语言特定指南

### TypeScript / JavaScript

```typescript
// SIMPLIFY: 不必要的 async 包装
// Before
async function getUser(id: string): Promise<User> {
  return await userService.findById(id);
}
// After
function getUser(id: string): Promise<User> {
  return userService.findById(id);
}

// SIMPLIFY: 冗长的条件赋值
// Before
let displayName: string;
if (user.nickname) {
  displayName = user.nickname;
} else {
  displayName = user.fullName;
}
// After
const displayName = user.nickname || user.fullName;

// SIMPLIFY: 手动构建数组
// Before
const activeUsers: User[] = [];
for (const user of users) {
  if (user.isActive) {
    activeUsers.push(user);
  }
}
// After
const activeUsers = users.filter((user) => user.isActive);

// SIMPLIFY: 冗余的布尔返回
// Before
function isValid(input: string): boolean {
  if (input.length > 0 && input.length < 100) {
    return true;
  }
  return false;
}
// After
function isValid(input: string): boolean {
  return input.length > 0 && input.length < 100;
}
```

### Python

```python
# SIMPLIFY: 冗长的字典构建
# Before
result = {}
for item in items:
    result[item.id] = item.name
# After
result = {item.id: item.name for item in items}

# SIMPLIFY: 带提前返回的嵌套条件
# Before
def process(data):
    if data is not None:
        if data.is_valid():
            if data.has_permission():
                return do_work(data)
            else:
                raise PermissionError("无权限")
        else:
            raise ValueError("无效数据")
    else:
        raise TypeError("数据为 None")
# After
def process(data):
    if data is None:
        raise TypeError("数据为 None")
    if not data.is_valid():
        raise ValueError("无效数据")
    if not data.has_permission():
        raise PermissionError("无权限")
    return do_work(data)
```

### React / JSX

```tsx
// SIMPLIFY: 冗长的条件渲染
// Before
function UserBadge({ user }: Props) {
  if (user.isAdmin) {
    return <Badge variant="admin">Admin</Badge>;
  } else {
    return <Badge variant="default">User</Badge>;
  }
}
// After
function UserBadge({ user }: Props) {
  const variant = user.isAdmin ? 'admin' : 'default';
  const label = user.isAdmin ? 'Admin' : 'User';
  return <Badge variant={variant}>{label}</Badge>;
}

// SIMPLIFY: 通过中间组件的 prop 下钻
// Before — 考虑 context 或组合是否能更好地解决这个问题。
// 这是一个判断性决策——标记它，不要自动重构。
```

## 常见借口

| 借口 | 现实 |
|---|---|
| "它能跑，不需要动它" | 能跑但难读的代码，在出问题时也难修复。现在简化会为未来的每次变更节省时间。 |
| "行数越少总是越简单" | 一行嵌套三元表达式并不比五行的 if/else 更简单。简单是关于理解速度，而非行数统计。 |
| "我就顺便快速简化一下这段无关代码" | 无范围的简化会产生嘈杂的 diff，并可能在你不打算变更的代码中引发回归。保持专注。 |
| "类型让它自文档化" | 类型文档化结构，而非意图。一个好名字的函数比类型签名更好地解释了*为什么*。 |
| "这个抽象以后可能会有用" | 不要保留推测性抽象。如果现在不用，它就是没有价值的复杂性。删除它，需要时再加回来。 |
| "原始作者肯定有理由" | 也许。检查 git blame——应用切斯特顿之篱。但累积的复杂性常常没有理由；它只是压力下迭代的残渣。 |
| "我一边加功能一边重构" | 将重构与功能工作分开。混合变更更难审查、回滚和在历史中理解。 |

## 警示信号

- 需要修改测试才能通过的简化（你很可能改变了行为）
- "简化"后的代码比原版更长、更难跟踪
- 重命名东西以匹配你的偏好而非项目约定
- 删除错误处理因为"这让代码更干净"
- 简化你没有完全理解的代码
- 将许多简化批处理进一个难以审查的大提交
- 在未被要求的情况下重构当前任务范围之外的代码

## 验证

完成一轮简化后：

- [ ] 所有现有测试在不修改的情况下通过
- [ ] 构建成功，没有新警告
- [ ] Linter/formatter 通过（无风格回归）
- [ ] 每个简化都是可审查的增量变更
- [ ] Diff 干净——没有混入无关变更
- [ ] 简化后的代码遵循项目约定（对照 CLAUDE.md 或等效文件检查过）
- [ ] 没有错误处理被移除或削弱
- [ ] 没有死代码遗留（未使用的导入、不可到达的分支）
- [ ] 队友或审查 agent 会批准该变更是一个净改进
