# 根因追踪

## 概述

Bug 经常在调用栈深处暴露（`git init` 在错误目录执行、文件创建在错误位置、数据库以错误路径打开）。你的直觉是在报错的地方修复，但那只是在治标。

**核心原则：** 沿调用链向上回溯，直到找到最初的触发点，然后在源头修复。

## 何时使用

```mermaid
flowchart TD
    A{"Bug 出现在调用栈深处？"}
    B{"能向上回溯吗？"}
    C["在症状处修复"]
    D["追踪到最初触发点"]
    E["更好：同时添加纵深防御"]

    A -->|yes| B
    B -->|yes| D
    B -->|"no - 走不通"| C
    D --> E
```

**使用场景：**
- 错误发生在执行深处（不在入口点）
- 堆栈跟踪显示很长的调用链
- 不清楚无效数据的来源
- 需要找到哪个测试/代码触发了问题

## 追踪过程

### 1. 观察症状
```
Error: git init failed in ~/project/packages/core
```

### 2. 找到直接原因
**什么代码直接导致了这个错误？**
```typescript
await execFileAsync('git', ['init'], { cwd: projectDir });
```

### 3. 问：谁调用了它？
```typescript
WorktreeManager.createSessionWorktree(projectDir, sessionId)
  → called by Session.initializeWorkspace()
  → called by Session.create()
  → called by test at Project.create()
```

### 4. 继续向上追踪
**传入了什么值？**
- `projectDir = ''`（空字符串！）
- 空字符串作为 `cwd` 会解析为 `process.cwd()`
- 那就是源码目录！

### 5. 找到最初触发点
**空字符串从哪来的？**
```typescript
const context = setupCoreTest(); // Returns { tempDir: '' }
Project.create('name', context.tempDir); // Accessed before beforeEach!
```

## 添加堆栈跟踪

当你无法手动追踪时，添加检测代码：

```typescript
// 在出问题的操作之前
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

**关键：** 在测试中使用 `console.error()`（不要用 logger - 可能不会显示）

**运行并捕获：**
```bash
npm test 2>&1 | grep 'DEBUG git init'
```

**分析堆栈跟踪：**
- 查找测试文件名
- 找到触发调用的行号
- 识别模式（同一个测试？同一个参数？）

## 找到哪个测试造成了污染

如果某些东西在测试期间出现，但你不知道是哪个测试：

使用本目录中的二分查找脚本 `find-polluter.sh`：

```bash
./find-polluter.sh '.git' 'src/**/*.test.ts'
```

逐个运行测试，在第一个污染者处停止。参见脚本了解用法。

## 真实案例：空的 projectDir

**症状：** `.git` 创建在 `packages/core/`（源码目录）中

**追踪链：**
1. `git init` 在 `process.cwd()` 中运行 ← 空的 cwd 参数
2. WorktreeManager 被传入空的 projectDir
3. Session.create() 传入了空字符串
4. 测试在 beforeEach 之前访问了 `context.tempDir`
5. setupCoreTest() 初始时返回 `{ tempDir: '' }`

**根因：** 顶层变量初始化时访问了空值

**修复：** 将 tempDir 改为 getter，在 beforeEach 之前访问时抛出异常

**同时添加了纵深防御：**
- 第 1 层：Project.create() 验证目录
- 第 2 层：WorkspaceManager 验证非空
- 第 3 层：NODE_ENV 守卫拒绝在 tmpdir 之外执行 git init
- 第 4 层：git init 之前的堆栈跟踪日志

## 关键原则

```mermaid
flowchart TD
    A(["找到直接原因"])
    B{"能向上追踪一层吗？"}
    C["向上回溯"]
    D{"这是源头吗？"}
    E["在源头修复"]
    F["在每一层添加验证"]
    G((("Bug 不可能再发生"))
    H{{"绝不只修症状"}}

    A --> B
    B -->|yes| C
    B -->|no| H
    C --> D
    D -->|"no - 继续追踪"| C
    D -->|yes| E
    E --> F
    F --> G
```

**绝不在错误出现的地方直接修复。** 向上回溯找到最初触发点。

## 堆栈跟踪技巧

**在测试中：** 使用 `console.error()` 而非 logger - logger 可能被抑制
**在操作之前：** 在危险操作之前记录日志，而不是在失败之后
**包含上下文：** 目录、cwd、环境变量、时间戳
**捕获堆栈：** `new Error().stack` 显示完整调用链

## 实际效果

来自调试会话（2025-10-03）：
- 通过 5 层追踪找到根因
- 在源头修复（getter 验证）
- 添加了 4 层防御
- 1847 个测试通过，零污染
