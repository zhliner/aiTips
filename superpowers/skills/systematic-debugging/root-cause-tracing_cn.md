# 根因追溯

## 概述

Bug 往往在调用栈深处显现（git init 在错误目录执行、文件在错误位置创建、数据库以错误路径打开）。你的直觉是在错误出现的地方修复，但那是在治标。

**核心原则：** 通过调用链向后追溯，直到找到原始触发点，然后在源头修复。

## 适用场景

```mermaid
flowchart TD
    A{Bug 出现在调用栈深处？}
    B{能否向后追溯？}
    C[在症状点修复]
    D[追溯到原始触发点]
    E[更好：同时添加纵深防御]

    A -->|是| B
    B -->|是| D
    B -->|否 - 死路| C
    D --> E
```

**适用情况：**
- 错误发生在执行深处（而非入口点）
- 堆栈跟踪显示长调用链
- 不清楚无效数据的来源
- 需要找出是哪个测试/代码触发了问题

## 追溯过程

### 1. 观察症状
```
Error: git init failed in ~/project/packages/core
```

### 2. 找到直接原因
**是哪段代码直接导致了它？**
```typescript
await execFileAsync('git', ['init'], { cwd: projectDir });
```

### 3. 追问：是谁调用了它？
```typescript
WorktreeManager.createSessionWorktree(projectDir, sessionId)
  → 被 Session.initializeWorkspace() 调用
  → 被 Session.create() 调用
  → 被测试中的 Project.create() 调用
```

### 4. 持续向上追溯
**传入的是什么值？**
- `projectDir = ''`（空字符串！）
- 空字符串作为 `cwd` 解析为 `process.cwd()`
- 那就是源代码目录！

### 5. 找到原始触发点
**空字符串是从哪里来的？**
```typescript
const context = setupCoreTest(); // 返回 { tempDir: '' }
Project.create('name', context.tempDir); // 在 beforeEach 之前访问！
```

## 添加堆栈跟踪

当你无法手动追溯时，添加工具：

```typescript
// 在有问题的操作之前
async function gitInit(directory: string) {
  const stack = new Error().stack;
  console.error('DEBUG git init:', {
    directory,
    cwd: process.cwd(),
    nodeEnv: process.env.NODE_ENV,
    stack,
  });

  await execFileAsync('git', ['init'], { cwd: directory });
}
```

**关键：** 在测试中使用 `console.error()`（不要用 logger — 可能不显示）

**运行并捕获：**
```bash
npm test 2>&1 | grep 'DEBUG git init'
```

**分析堆栈跟踪：**
- 查找测试文件名
- 找到触发调用的行号
- 识别模式（同一个测试？同一个参数？）

## 找出是哪个测试造成了污染

如果在测试期间出现了某物但你不知道是哪个测试：

使用本目录下的二分查找脚本 `find-polluter.sh`：

```bash
./find-polluter.sh '.git' 'src/**/*.test.ts'
```

逐个运行测试，在第一个污染者处停止。用法参见脚本本身。

## 真实案例：空的 projectDir

**症状：** `.git` 在 `packages/core/` 中被创建（源代码目录）

**追溯链：**
1. `git init` 在 `process.cwd()` 中执行 ← 空的 cwd 参数
2. WorktreeManager 被传入了空的 projectDir
3. Session.create() 传入了空字符串
4. 测试在 beforeEach 之前访问了 `context.tempDir`
5. setupCoreTest() 初始返回 `{ tempDir: '' }`

**根因：** 顶层变量初始化访问了空值

**修复：** 将 tempDir 改为 getter，在 beforeEach 之前访问时抛出异常

**同时添加了纵深防御：**
- 第 1 层：Project.create() 验证目录
- 第 2 层：WorkspaceManager 验证非空
- 第 3 层：NODE_ENV 守卫拒绝在 tmpdir 之外执行 git init
- 第 4 层：git init 之前的堆栈跟踪日志

## 核心原则

```mermaid
flowchart TD
    A([找到直接原因])
    B{能否再向上一级追溯？}
    C[向后追溯]
    D{这是源头吗？}
    E[在源头修复]
    F[在每一层添加验证]
    G([Bug 不可能发生])
    H[绝不只修复症状点]

    A --> B
    B -->|是| C
    B -->|否| H
    C --> D
    D -->|否 - 继续| C
    D -->|是| E
    E --> F
    F --> G
```

**绝不只在错误出现的位置修复。** 向后追溯以找到原始触发点。

## 堆栈跟踪技巧

**在测试中：** 使用 `console.error()` 而非 logger — logger 可能被抑制
**在操作之前：** 在危险操作之前记录日志，而非在失败之后
**包含上下文：** 目录、cwd、环境变量、时间戳
**捕获堆栈：** `new Error().stack` 显示完整的调用链

## 实际效果

来自调试会话（2025-10-03）：
- 通过 5 级追溯找到了根因
- 在源头修复（getter 验证）
- 添加了 4 层防御
- 1847 个测试通过，零污染
