# 纵深防御验证（Defense-in-Depth Validation）

## 概述

当你修复了一个由无效数据引起的 Bug 时，在一个地方添加验证感觉就足够了。但是，这一个检查可能被不同的代码路径、重构或 mock 绕过。

**核心原则：** 在数据经过的每一层都进行验证。让 Bug 在结构上不可能发生。

## 为什么需要多层

单层验证："我们修好了这个 Bug"
多层防御："我们让这个 Bug 不可能再发生"

不同层级捕获不同的情况：
- 入口验证捕获大多数 Bug
- 业务逻辑捕获边界情况
- 环境守卫防止特定上下文的危险
- Debug 日志在其他层失效时提供帮助

## 四层防御

### 第 1 层：入口点验证
**目的：** 在 API 边界拒绝明显无效的输入

```typescript
function createProject(name: string, workingDirectory: string) {
  if (!workingDirectory || workingDirectory.trim() === '') {
    throw new Error('workingDirectory cannot be empty');
  }
  if (!existsSync(workingDirectory)) {
    throw new Error(`workingDirectory does not exist: ${workingDirectory}`);
  }
  if (!statSync(workingDirectory).isDirectory()) {
    throw new Error(`workingDirectory is not a directory: ${workingDirectory}`);
  }
  // ... proceed
}
```

### 第 2 层：业务逻辑验证
**目的：** 确保数据对该操作而言是合理的

```typescript
function initializeWorkspace(projectDir: string, sessionId: string) {
  if (!projectDir) {
    throw new Error('projectDir required for workspace initialization');
  }
  // ... proceed
}
```

### 第 3 层：环境守卫
**目的：** 防止在特定上下文中执行危险操作

```typescript
async function gitInit(directory: string) {
  // 在测试中，拒绝在临时目录之外执行 git init
  if (process.env.NODE_ENV === 'test') {
    const normalized = normalize(resolve(directory));
    const tmpDir = normalize(resolve(tmpdir()));

    if (!normalized.startsWith(tmpDir)) {
      throw new Error(
        `Refusing git init outside temp dir during tests: ${directory}`
      );
    }
  }
  // ... proceed
}
```

### 第 4 层：Debug Instrumentation
**目的：** 捕获上下文以供事后分析

```typescript
async function gitInit(directory: string) {
  const stack = new Error().stack;
  logger.debug('About to git init', {
    directory,
    cwd: process.cwd(),
    stack,
  });
  // ... proceed
}
```

## 应用模式

当你发现一个 Bug 时：

1. **追踪数据流**——无效值从哪里来？在哪里被使用？
2. **映射所有检查点**——列出数据经过的每一个点
3. **在每一层添加验证**——入口、业务、环境、debug
4. **测试每一层**——尝试绕过第 1 层，验证第 2 层能捕获它

## 来自实践的例子

Bug：空的 `projectDir` 导致 `git init` 在源代码中执行

**数据流：**
1. 测试设置 → 空字符串
2. `Project.create(name, '')`
3. `WorkspaceManager.createWorkspace('')`
4. `git init` 在 `process.cwd()` 中执行

**添加的四层防御：**
- 第 1 层：`Project.create()` 验证不为空/存在/可写
- 第 2 层：`WorkspaceManager` 验证 projectDir 不为空
- 第 3 层：`WorktreeManager` 在测试中拒绝在 tmpdir 之外执行 git init
- 第 4 层：git init 前的 stack trace 日志

**结果：** 全部 1847 个测试通过，Bug 无法复现

## 关键洞察

所有四层都是必要的。在测试过程中，每一层都捕获了其他层遗漏的 Bug：
- 不同的代码路径绕过了入口验证
- Mock 绕过了业务逻辑检查
- 不同平台的边界情况需要环境守卫
- Debug 日志识别出了结构性的误用

**不要停在一个验证点上。** 在每一层都添加检查。
