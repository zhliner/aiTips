---
name: test-driven-development
description: 在实现任何功能或修复 bug 时，在编写实现代码之前使用
---

# 测试驱动开发（Test-Driven Development, TDD）

## 概述

先写测试。看它失败。写最少代码使其通过。

**核心原则：** 如果你没有看到测试失败，你就不知道它是否测试了正确的东西。

**违反规则的字面意思就是违反规则的精神。**

## 何时使用

**始终：**
- 新功能
- Bug 修复
- 重构
- 行为变更

**例外（需征求人类伙伴同意）：**
- 一次性原型
- 生成的代码
- 配置文件

想着"就这一次跳过 TDD"？停下来。那是合理化。

## 铁律

```
没有先写失败测试，就不能写生产代码
```

在测试之前写了代码？删掉它。从头开始。

**没有例外：**
- 不要保留它作为"参考"
- 不要在写测试时"适配"它
- 不要看它
- 删除就是删除

从测试出发全新实现。句号。

## 红-绿-重构

```dot
digraph tdd_cycle {
    rankdir=LR;
    red [label="RED\nWrite failing test", shape=box, style=filled, fillcolor="#ffcccc"];
    verify_red [label="Verify fails\ncorrectly", shape=diamond];
    green [label="GREEN\nMinimal code", shape=box, style=filled, fillcolor="#ccffcc"];
    verify_green [label="Verify passes\nAll green", shape=diamond];
    refactor [label="REFACTOR\nClean up", shape=box, style=filled, fillcolor="#ccccff"];
    next [label="Next", shape=ellipse];

    red -> verify_red;
    verify_red -> green [label="yes"];
    verify_red -> red [label="wrong\nfailure"];
    green -> verify_green;
    verify_green -> refactor [label="yes"];
    verify_green -> green [label="no"];
    refactor -> verify_green [label="stay\ngreen"];
    verify_green -> next;
    next -> red;
}
```

### RED - 编写失败测试

编写一个最小的测试，展示期望的行为。

<Good>
```typescript
test('retries failed operations 3 times', async () => {
  let attempts = 0;
  const operation = () => {
    attempts++;
    if (attempts < 3) throw new Error('fail');
    return 'success';
  };

  const result = await retryOperation(operation);

  expect(result).toBe('success');
  expect(attempts).toBe(3);
});
```
清晰的名称，测试真实行为，只测一件事
</Good>

<Bad>
```typescript
test('retry works', async () => {
  const mock = jest.fn()
    .mockRejectedValueOnce(new Error())
    .mockRejectedValueOnce(new Error())
    .mockResolvedValueOnce('success');
  await retryOperation(mock);
  expect(mock).toHaveBeenCalledTimes(3);
});
```
模糊的名称，测试的是 mock 而不是代码
</Bad>

**要求：**
- 一个行为
- 清晰的名称
- 真实代码（除非不得已才用 mock）

### 验证 RED - 看它失败

**强制执行。绝不跳过。**

```bash
npm test path/to/test.test.ts
```

确认：
- 测试失败（不是报错）
- 失败信息符合预期
- 因为功能缺失而失败（不是因为拼写错误）

**测试通过了？** 你在测试已有的行为。修改测试。

**测试报错了？** 修复错误，重新运行直到它正确地失败。

### GREEN - 最少代码

编写最简单的代码使测试通过。

<Good>
```typescript
async function retryOperation<T>(fn: () => Promise<T>): Promise<T> {
  for (let i = 0; i < 3; i++) {
    try {
      return await fn();
    } catch (e) {
      if (i === 2) throw e;
    }
  }
  throw new Error('unreachable');
}
```
刚好够通过测试
</Good>

<Bad>
```typescript
async function retryOperation<T>(
  fn: () => Promise<T>,
  options?: {
    maxRetries?: number;
    backoff?: 'linear' | 'exponential';
    onRetry?: (attempt: number) => void;
  }
): Promise<T> {
  // YAGNI
}
```
过度工程化
</Bad>

不要添加功能、重构其他代码或超出测试范围做"改进"。

### 验证 GREEN - 看它通过

**强制执行。**

```bash
npm test path/to/test.test.ts
```

确认：
- 测试通过
- 其他测试仍然通过
- 输出干净（没有错误、警告）

**测试失败了？** 修改代码，而不是测试。

**其他测试失败了？** 立即修复。

### REFACTOR - 清理

仅在绿色通过之后：
- 消除重复
- 改进命名
- 提取辅助函数

保持测试通过。不要添加行为。

### 重复

为下一个功能编写下一个失败测试。

## 好的测试

| 质量 | 好的 | 差的 |
|------|------|------|
| **最小化** | 一件事。名称中有"and"？拆分它。 | `test('validates email and domain and whitespace')` |
| **清晰** | 名称描述行为 | `test('test1')` |
| **展示意图** | 展示期望的 API | 模糊了代码应该做什么 |

## 为什么顺序很重要

**"我先写完代码再写测试来验证它能用"**

在代码之后写的测试会立即通过。立即通过什么也证明不了：
- 可能测试了错误的东西
- 可能测试的是实现而不是行为
- 可能遗漏了你忘记的边界情况
- 你从未看到它捕获过 bug

测试优先迫使你看到测试失败，证明它确实在测试某些东西。

**"我已经手动测试了所有边界情况"**

手动测试是临时的。你以为你测试了所有情况，但是：
- 没有测试记录
- 代码更改后无法重新运行
- 在压力下容易遗漏
- "我试的时候能用" ≠ 全面

自动化测试是系统化的。它们每次以相同方式运行。

**"删除 X 小时的工作太浪费了"**

沉没成本谬误。时间已经过去了。你现在的选择：
- 删除并用 TDD 重写（再花 X 小时，高信心）
- 保留并事后添加测试（30 分钟，低信心，可能有 bug）

"浪费"的是保留你无法信任的代码。没有真正测试的可运行代码就是技术债。

**"TDD 是教条的，务实意味着灵活"**

TDD 就是务实的：
- 在提交前发现 bug（比事后调试更快）
- 防止回归（测试立即捕获破坏）
- 记录行为（测试展示如何使用代码）
- 使重构成为可能（放心修改，测试捕获破坏）

"务实"的捷径 = 在生产环境调试 = 更慢。

**"事后测试达到同样目标——重要的是精神不是仪式"**

不对。事后测试回答"这做了什么？"测试优先回答"这应该做什么？"

事后测试受你的实现偏见影响。你测试的是你构建了什么，而不是需求是什么。你验证的是记住的边界情况，而不是发现的边界情况。

测试优先迫使在实现之前发现边界情况。事后测试验证你记住了所有情况（你没有）。

30 分钟的事后测试 ≠ TDD。你得到了覆盖率，失去了测试有效的证明。

## 常见的自我合理化

| 借口 | 现实 |
|------|------|
| "太简单了不需要测试" | 简单代码也会出错。测试只需 30 秒。 |
| "我之后再测试" | 立即通过的测试什么也证明不了。 |
| "事后测试达到同样目标" | 事后测试 = "这做了什么？"测试优先 = "这应该做什么？" |
| "已经手动测试过了" | 临时 ≠ 系统化。没有记录，无法重新运行。 |
| "删除 X 小时的工作太浪费了" | 沉没成本谬误。保留未验证的代码才是技术债。 |
| "保留作为参考，先写测试" | 你会去适配它。那就是事后测试。删除就是删除。 |
| "需要先探索一下" | 可以。丢弃探索结果，用 TDD 从头开始。 |
| "测试难写 = 设计不清晰" | 听从测试。难以测试 = 难以使用。 |
| "TDD 会拖慢我" | TDD 比调试快。务实 = 测试优先。 |
| "手动测试更快" | 手动测试无法证明边界情况。每次更改你都得重新测试。 |
| "现有代码没有测试" | 你在改进它。为现有代码添加测试。 |

## 危险信号——停下来从头开始

- 在测试之前写代码
- 在实现之后写测试
- 测试立即通过
- 无法解释为什么测试失败
- 测试"以后"再补
- 合理化"就这一次"
- "我已经手动测试过了"
- "事后测试达到同样目的"
- "重要的是精神不是仪式"
- "保留作为参考"或"适配现有代码"
- "已经花了 X 小时，删除太浪费了"
- "TDD 是教条的，我在务实"
- "这次不一样因为……"

**以上所有情况意味着：删除代码。用 TDD 从头开始。**

## 示例：Bug 修复

**Bug：** 空邮箱被接受了

**RED**
```typescript
test('rejects empty email', async () => {
  const result = await submitForm({ email: '' });
  expect(result.error).toBe('Email required');
});
```

**验证 RED**
```bash
$ npm test
FAIL: expected 'Email required', got undefined
```

**GREEN**
```typescript
function submitForm(data: FormData) {
  if (!data.email?.trim()) {
    return { error: 'Email required' };
  }
  // ...
}
```

**验证 GREEN**
```bash
$ npm test
PASS
```

**REFACTOR**
如有需要，提取验证逻辑以支持多个字段。

## 验证检查清单

在标记工作完成之前：

- [ ] 每个新函数/方法都有测试
- [ ] 在实现之前观看了每个测试失败
- [ ] 每个测试因预期原因失败（功能缺失，不是拼写错误）
- [ ] 为每个测试编写了最少代码使其通过
- [ ] 所有测试通过
- [ ] 输出干净（没有错误、警告）
- [ ] 测试使用真实代码（只在不可避免时使用 mock）
- [ ] 覆盖了边界情况和错误处理

无法勾选所有选项？你跳过了 TDD。从头开始。

## 遇到困难时

| 问题 | 解决方案 |
|------|---------|
| 不知道如何测试 | 写出期望的 API。先写断言。征求人类伙伴意见。 |
| 测试太复杂 | 设计太复杂。简化接口。 |
| 必须 mock 所有东西 | 代码耦合太紧。使用依赖注入。 |
| 测试设置庞大 | 提取辅助函数。仍然复杂？简化设计。 |

## 与调试的整合

发现 bug？编写失败的测试复现它。遵循 TDD 循环。测试证明修复并防止回归。

绝不在没有测试的情况下修复 bug。

## 测试反模式

添加 mock 或测试工具时，阅读 @testing-anti-patterns.md 以避免常见陷阱：
- 测试 mock 行为而不是真实行为
- 在生产类中添加仅用于测试的方法
- 在不理解依赖的情况下使用 mock

## 最终规则

```
生产代码 → 有测试且测试先失败过
否则 → 不是 TDD
```

未经人类伙伴许可不得有例外。
