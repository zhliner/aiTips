# Defense-in-Depth Validation（纵深防御验证）

## Overview（概述）

当你修复由无效数据引起的错误时，在一个地方添加验证感觉足够。但单一检查可能被不同的代码路径、重构或 mock 绕过。

**核心原则：** 在数据经过的每一层都验证。使错误在结构上不可能发生。

## Why Multiple Layers（为什么需要多层）

单一验证："我们修复了错误"
多层："我们使错误不可能"

不同层捕获不同情况：
- 入口验证捕获大多数错误
- 业务逻辑捕获边界情况
- 环境防护防止特定上下文的危险
- 调试日志在其他层失败时提供帮助

## The Four Layers（四层）

### Layer 1: Entry Point Validation（层 1：入口点验证）
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

### Layer 2: Business Logic Validation（层 2：业务逻辑验证）
**目的：** 确保数据对此操作有意义

```typescript
function initializeWorkspace(projectDir: string, sessionId: string) {
  if (!projectDir) {
    throw new Error('projectDir required for workspace initialization');
  }
  // ... proceed
}
```

### Layer 3: Environment Guards（层 3：环境防护）
**目的：** 在特定上下文中防止危险操作

```typescript
async function gitInit(directory: string) {
  // 在测试中，拒绝在临时目录之外 git init
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

### Layer 4: Debug Instrumentation（层 4：调试工具）
**目的：** 为取证捕获上下文

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

## Applying the Pattern（应用模式）

当你发现错误时：

1. **追踪数据流** - 错误值源自哪里？在哪里使用？
2. **映射所有检查点** - 列出数据经过的每个点
3. **在每层添加验证** - 入口、业务、环境、调试
4. **测试每层** - 尝试绕过层 1，验证层 2 捕获它

## Example from Session（会话示例）

错误：空 `projectDir` 导致在源代码中 `git init`

**数据流：**
1. 测试设置 → 空字符串
2. `Project.create(name, '')`
3. `WorkspaceManager.createWorkspace('')`
4. `git init` 在 `process.cwd()` 中运行

**添加的四层：**
- 层 1：`Project.create()` 验证非空/存在/可写
- 层 2：`WorkspaceManager` 验证 projectDir 非空
- 层 3：`WorktreeManager` 在测试中拒绝在 tmpdir 之外 git init
- 层 4：git init 之前的栈跟踪日志

**结果：** 所有 1847 个测试通过，错误不可能复现

## Key Insight（关键见解）

所有四层都是必需的。在测试期间，每层捕获其他层遗漏的错误：
- 不同的代码路径绕过入口验证
- Mock 绕过业务逻辑检查
- 不同平台上的边界情况需要环境防护
- 调试日志识别结构误用

**不要停在一个验证点。** 在每层添加检查。
