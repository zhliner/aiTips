---
name: code-simplification
description: 简化代码以提升清晰度。适用于在不改变行为的前提下重构代码以提高可读性。适用于代码能正常运行，但可读性、可维护性或可扩展性不如预期的场景。适用于审查已积累不必要复杂性的代码。
---

# 代码简化（Code Simplification）

> 灵感来自 [Claude Code Simplifier 插件](https://github.com/anthropics/claude-plugins-official/blob/main/plugins/code-simplifier/agents/code-simplifier.md)。在此改编为模型无关、流程驱动的技能，适用于任何 AI 编程代理。

## 概述

通过降低复杂性来简化代码，同时严格保留原有行为。目标不是减少代码行数——而是让代码更容易阅读、理解、修改和调试。每一次简化都必须通过一个简单的检验："新加入团队的成员能否比看原始代码更快地理解它？"

## 适用时机

- 功能已实现且测试通过，但实现方式感觉过于笨重时
- 代码审查中发现可读性或复杂度问题时
- 遇到深层嵌套逻辑、过长函数或含义不明的命名时
- 重构在时间压力下编写的代码时
- 合并分散在多个文件中的相关逻辑时
- 合并引入了重复或不一致的代码后

**不适用时机：**

- 代码已经整洁可读——不要为了简化而简化
- 你还不理解代码的功能——先理解，再简化
- 代码对性能有严格要求，而"更简单"的版本会明显变慢
- 你打算整体重写该模块——简化即将被丢弃的代码是浪费精力

## 五大原则

### 1. 严格保留原有行为

不要改变代码的功能——只改变它的表达方式。所有输入、输出、副作用、错误处理和边界情况必须完全一致。如果你不确定某次简化是否保留了原有行为，就不要执行它。

```
每次修改前都要问自己：
→ 对于每个输入，输出是否相同？
→ 错误处理行为是否一致？
→ 副作用和执行顺序是否保留？
→ 所有现有测试是否无需修改即可通过？
```

### 2. 遵循项目约定

简化意味着让代码与代码库更加一致，而不是强加外部偏好。在简化之前：

```
1. 阅读 CLAUDE.md / 项目约定文档
2. 研究相邻代码如何处理类似模式
3. 匹配项目的风格：
   - 导入顺序与模块系统
   - 函数声明风格
   - 命名约定
   - 错误处理模式
   - 类型标注深度
```

破坏项目一致性的简化不是简化——是折腾。

### 3. 清晰优先于巧妙

当紧凑版本需要停顿思考才能理解时，显式的代码优于紧凑的代码。

```typescript
// 不清晰：密集的三元表达式链
const label = isNew ? 'New' : isUpdated ? 'Updated' : isArchived ? 'Archived' : 'Active';

// 清晰：可读的映射逻辑
function getStatusLabel(item: Item): string {
  if (item.isNew) return 'New';
  if (item.isUpdated) return 'Updated';
  if (item.isArchived) return 'Archived';
  return 'Active';
}
```

```typescript
// 不清晰：链式 reduce 内联逻辑
const result = items.reduce((acc, item) => ({
  ...acc,
  [item.id]: { ...acc[item.id], count: (acc[item.id]?.count ?? 0) + 1 }
}), {});

// 清晰：命名中间步骤
const countById = new Map<string, number>();
for (const item of items) {
  countById.set(item.id, (countById.get(item.id) ?? 0) + 1);
}
```

### 4. 保持平衡

简化有一个失败模式：过度简化。注意以下陷阱：

- **过度内联** — 删除了为概念命名的辅助函数，反而让调用处更难阅读
- **合并不相关的逻辑** — 将两个简单函数合并成一个复杂函数并不更简单
- **移除"不必要"的抽象** — 有些抽象是为了可扩展性或可测试性而存在，并非为了增加复杂度
- **以行数为优化目标** — 更少的行数不是目标；更容易理解才是

### 5. 限定修改范围

默认只简化最近修改过的代码。除非明确要求扩大范围，否则避免顺手重构不相关的代码。超出范围的简化会在 diff 中引入噪音，并带来意外的回归风险。

## 简化流程

### 第 1 步：先理解再动手（Chesterton's Fence）

在修改或删除任何内容之前，先理解它为什么存在。这就是 Chesterton's Fence（切斯特顿之篱）：如果你在路边看到一道篱笆却不理解它为什么在那里，不要拆除它。先理解原因，再判断该原因是否仍然成立。

```
简化之前，回答以下问题：
- 这段代码的职责是什么？
- 谁调用了它？它调用了什么？
- 有哪些边界情况和错误路径？
- 是否有测试定义了预期行为？
- 它当初为什么这样写？（性能？平台限制？历史原因？）
- 查看 git blame：这段代码最初的上下文是什么？
```

如果你无法回答这些问题，说明你还没准备好简化。先阅读更多上下文。

### 第 2 步：识别简化机会

扫描以下模式——每个都是具体的信号，而非模糊的感觉：

**结构性复杂度：**

| 模式 | 信号 | 简化方式 |
|------|------|----------|
| 深层嵌套（3 层以上） | 控制流难以跟踪 | 将条件提取为 guard clause 或辅助函数 |
| 过长函数（50 行以上） | 承担多个职责 | 拆分为职责单一、命名清晰的函数 |
| 嵌套三元表达式 | 需要心算栈来解析 | 替换为 if/else 链、switch 或查找对象 |
| 布尔参数标记 | `doThing(true, false, true)` | 替换为选项对象或独立函数 |
| 重复条件判断 | 相同 `if` 检查出现在多处 | 提取为命名良好的谓词函数 |

**命名与可读性：**

| 模式 | 信号 | 简化方式 |
|------|------|----------|
| 泛化命名 | `data`、`result`、`temp`、`val`、`item` | 重命名以描述内容：`userProfile`、`validationErrors` |
| 缩写命名 | `usr`、`cfg`、`btn`、`evt` | 使用完整单词，除非缩写已成通用惯例（`id`、`url`、`api`） |
| 误导性命名 | 名为 `get` 的函数同时修改了状态 | 重命名以反映实际行为 |
| 解释"做了什么"的注释 | `count++` 上方的 `// increment counter` | 删除注释——代码本身已经足够清晰 |
| 解释"为什么"的注释 | `// Retry because the API is flaky under load` | 保留——它们传达了代码无法表达的意图 |

**冗余：**

| 模式 | 信号 | 简化方式 |
|------|------|----------|
| 重复逻辑 | 相同 5 行以上代码出现在多处 | 提取为共享函数 |
| 死代码 | 不可达分支、未使用变量、被注释的代码块 | 移除（确认确实无用后） |
| 不必要的抽象 | 未增加任何价值的包装层 | 内联包装层，直接调用底层函数 |
| 过度设计的模式 | 工厂的工厂、只有一个策略的策略模式 | 替换为简单直接的方式 |
| 冗余类型断言 | 断言为已推断出的类型 | 移除断言 |

### 第 3 步：增量应用修改

每次只做一项简化。每次修改后运行测试。**将重构变更与功能或 Bug 修复变更分开提交。** 一个同时包含重构和功能新增的 PR 实际上是两个 PR——拆分它们。

```
每次简化：
1. 执行修改
2. 运行测试套件
3. 如果测试通过 → 提交（或继续下一项简化）
4. 如果测试失败 → 回退并重新考虑
```

避免将多项简化批量合并为一次未经测试的修改。如果出了问题，你需要知道是哪项简化导致的。

**500 行规则：** 如果一次重构涉及超过 500 行修改，应投入自动化工具（codemod、sed 脚本、AST 转换），而不是手动修改。在这种规模下手动编辑容易出错，且审查起来令人疲惫。

### 第 4 步：验证结果

所有简化完成后，退后一步评估整体效果：

```
对比修改前后：
- 简化后的版本是否真的更容易理解？
- 是否引入了与代码库不一致的新模式？
- diff 是否干净且易于审查？
- 队友会批准这个变更吗？
```

如果"简化"后的版本更难理解或审查，就回退。不是每次简化尝试都能成功。

## 语言特定指南

### TypeScript / JavaScript

```typescript
// 简化：不必要的 async 包装
// 修改前
async function getUser(id: string): Promise<User> {
  return await userService.findById(id);
}
// 修改后
function getUser(id: string): Promise<User> {
  return userService.findById(id);
}

// 简化：冗长的条件赋值
// 修改前
let displayName: string;
if (user.nickname) {
  displayName = user.nickname;
} else {
  displayName = user.fullName;
}
// 修改后
const displayName = user.nickname || user.fullName;

// 简化：手动构建数组
// 修改前
const activeUsers: User[] = [];
for (const user of users) {
  if (user.isActive) {
    activeUsers.push(user);
  }
}
// 修改后
const activeUsers = users.filter((user) => user.isActive);

// 简化：冗余的布尔返回值
// 修改前
function isValid(input: string): boolean {
  if (input.length > 0 && input.length < 100) {
    return true;
  }
  return false;
}
// 修改后
function isValid(input: string): boolean {
  return input.length > 0 && input.length < 100;
}
```

### Python

```python
# 简化：冗长的字典构建
# 修改前
result = {}
for item in items:
    result[item.id] = item.name
# 修改后
result = {item.id: item.name for item in items}

# 简化：使用提前返回消除嵌套条件
# 修改前
def process(data):
    if data is not None:
        if data.is_valid():
            if data.has_permission():
                return do_work(data)
            else:
                raise PermissionError("No permission")
        else:
            raise ValueError("Invalid data")
    else:
        raise TypeError("Data is None")
# 修改后
def process(data):
    if data is None:
        raise TypeError("Data is None")
    if not data.is_valid():
        raise ValueError("Invalid data")
    if not data.has_permission():
        raise PermissionError("No permission")
    return do_work(data)
```

### React / JSX

```tsx
// 简化：冗长的条件渲染
// 修改前
function UserBadge({ user }: Props) {
  if (user.isAdmin) {
    return <Badge variant="admin">Admin</Badge>;
  } else {
    return <Badge variant="default">User</Badge>;
  }
}
// 修改后
function UserBadge({ user }: Props) {
  const variant = user.isAdmin ? 'admin' : 'default';
  const label = user.isAdmin ? 'Admin' : 'User';
  return <Badge variant={variant}>{label}</Badge>;
}

// 简化：通过中间组件逐层传递 props
// 修改前——考虑是否用 context 或组合模式更好地解决此问题。
// 这需要判断——标记出来，不要自动重构。
```

## 常见自我说服的借口

| 借口 | 现实 |
|------|------|
| "能跑就行，没必要动" | 难以阅读的能跑的代码，出 Bug 时同样难以修复。现在简化可以节省未来每次修改的时间。 |
| "行数越少越简单" | 一行的嵌套三元表达式并不比五行的 if/else 更简单。简洁关乎理解速度，而非行数。 |
| "我顺手把这个不相关的代码也简化一下" | 超出范围的简化会产生嘈杂的 diff，并可能在你无意修改的代码中引入回归。保持专注。 |
| "类型已经让它自文档化了" | 类型文档化的是结构，而非意图。命名良好的函数比类型签名更好地解释了*为什么*。 |
| "这个抽象以后可能有用" | 不要保留投机性的抽象。如果现在没用到，它就是没有价值的复杂度。移除它，需要时再加回来。 |
| "原作者这样写肯定有原因" | 也许吧。查看 git blame——运用 Chesterton's Fence。但积累的复杂性往往没有原因，它只是压力下迭代的残留物。 |
| "我边加功能边重构" | 将重构与功能开发分开。混合的变更更难审查、回退和在历史中理解。 |

## 危险信号

- 简化后需要修改测试才能通过（你可能改变了行为）
- "简化"后的代码比原始代码更长、更难理解
- 按个人偏好而非项目约定重命名
- 以"让代码更干净"为由移除错误处理
- 简化你不完全理解的代码
- 将多项简化批量合并为一次难以审查的大型提交
- 未经要求就重构当前任务范围之外的代码

## 验证清单

完成简化后：

- [ ] 所有现有测试无需修改即可通过
- [ ] 构建成功且无新增警告
- [ ] Linter / formatter 通过（无风格回归）
- [ ] 每项简化都是可审查的增量变更
- [ ] diff 干净——没有混入无关修改
- [ ] 简化后的代码遵循项目约定（对照 CLAUDE.md 或等效文档检查）
- [ ] 没有移除或削弱错误处理
- [ ] 没有遗留死代码（未使用的导入、不可达分支）
- [ ] 队友或审查代理会认为此变更是一项净改进
