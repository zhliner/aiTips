---
name: ci-cd-and-automation
description: 自动化 CI/CD pipeline 配置。在设置或修改构建与部署 pipeline 时使用。当你需要自动化质量门禁、在 CI 中配置测试运行器，或建立部署策略时使用。
---

# CI/CD 与自动化

## 概述

自动化质量门禁，确保没有任何变更在未通过测试、lint、类型检查和构建的情况下进入生产环境。CI/CD 是所有其他技能的执行机制——它能捕获人类和 agent 遗漏的问题，并在每一次变更中始终如一地执行检查。

**左移（Shift Left）：** 尽可能在 pipeline 的早期阶段发现问题。在 lint 阶段发现的 bug 只需几分钟修复；同样的 bug 在生产环境发现则需数小时。将检查前置——静态分析在测试之前，测试在预发布之前，预发布在生产之前。

**更快即更安全：** 更小的批次和更频繁的发布会降低风险，而非增加风险。包含 3 个变更的部署比包含 30 个变更的部署更容易调试。频繁发布能建立对发布流程本身的信心。

## 使用时机

- 为新项目搭建 CI pipeline
- 添加或修改自动化检查
- 配置部署 pipeline
- 当某次变更需要触发自动化验证时
- 调试 CI 失败

## 质量门禁 Pipeline

每次变更在合并前都需经过以下门禁：

```
Pull Request Opened
    │
    ▼
┌─────────────────┐
│   LINT CHECK     │  eslint, prettier
│   ↓ pass         │
│   TYPE CHECK     │  tsc --noEmit
│   ↓ pass         │
│   UNIT TESTS     │  jest/vitest
│   ↓ pass         │
│   BUILD          │  npm run build
│   ↓ pass         │
│   INTEGRATION    │  API/DB tests
│   ↓ pass         │
│   E2E (optional) │  Playwright/Cypress
│   ↓ pass         │
│   SECURITY AUDIT │  npm audit
│   ↓ pass         │
│   BUNDLE SIZE    │  bundlesize check
└─────────────────┘
    │
    ▼
  Ready for review
```

**不可跳过任何门禁。** 如果 lint 失败，修复 lint——不要禁用规则。如果测试失败，修复代码——不要跳过测试。

## GitHub Actions 配置

### 基础 CI Pipeline

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Type check
        run: npx tsc --noEmit

      - name: Test
        run: npm test -- --coverage

      - name: Build
        run: npm run build

      - name: Security audit
        run: npm audit --audit-level=high
```

### 包含数据库集成测试

```yaml
  integration:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: ci_user
          POSTGRES_PASSWORD: ${{ secrets.CI_DB_PASSWORD }}
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci
      - name: Run migrations
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: postgresql://ci_user:${{ secrets.CI_DB_PASSWORD }}@localhost:5432/testdb
      - name: Integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://ci_user:${{ secrets.CI_DB_PASSWORD }}@localhost:5432/testdb
```

> **注意：** 即使是仅用于 CI 的测试数据库，也应使用 GitHub Secrets 管理凭据，而非硬编码值。这有助于养成良好习惯，并防止测试凭据在其他场景中被意外复用。

### E2E 测试

```yaml
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci
      - name: Install Playwright
        run: npx playwright install --with-deps chromium
      - name: Build
        run: npm run build
      - name: Run E2E tests
        run: npx playwright test
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/
```

## 将 CI 失败反馈给 Agent

CI 与 AI agent 结合的强大之处在于反馈循环。当 CI 失败时：

```
CI 失败
    │
    ▼
复制失败输出
    │
    ▼
将其提供给 agent：
"CI pipeline 出现以下错误：
[粘贴具体错误信息]
请修复问题并在本地验证通过后再推送。"
    │
    ▼
Agent 修复 → 推送 → CI 再次运行
```

**关键模式：**

```
Lint 失败 → Agent 运行 `npm run lint --fix` 并提交
类型错误  → Agent 读取错误位置并修复类型
测试失败 → Agent 遵循 debugging-and-error-recovery 技能
构建错误 → Agent 检查配置和依赖
```

## 部署策略

### 预览部署

每个 PR 都应获得一个预览部署，用于手动测试：

```yaml
# 在 PR 时部署预览（Vercel/Netlify 等）
deploy-preview:
  runs-on: ubuntu-latest
  if: github.event_name == 'pull_request'
  steps:
    - uses: actions/checkout@v4
    - name: Deploy preview
      run: npx vercel --token=${{ secrets.VERCEL_TOKEN }}
```

### 功能开关（Feature Flags）

功能开关将部署与发布解耦。将未完成或有风险的功能置于开关之后，以便：

- **部署代码而不启用它。** 尽早合并到 main，准备好后再启用。
- **无需重新部署即可回滚。** 禁用开关，而不是回退代码。
- **灰度发布新功能。** 先对 1% 的用户启用，然后 10%，最后 100%。
- **运行 A/B 测试。** 比较有和没有该功能时的行为差异。

```typescript
// 简单的功能开关模式
if (featureFlags.isEnabled('new-checkout-flow', { userId })) {
  return renderNewCheckout();
}
return renderLegacyCheckout();
```

**开关的生命周期：** 创建 → 启用测试 → 灰度发布 → 全量上线 → 移除开关和废弃代码。长期存在的开关会变成技术债务——创建时就要设定清理日期。

### 分阶段发布

```
PR 合并到 main
    │
    ▼
  预发布环境部署（自动）
    │ 手动验证
    ▼
  生产环境部署（手动触发或在预发布后自动）
    │
    ▼
  监控错误（15 分钟观察窗口）
    │
    ├── 检测到错误 → 回滚
    └── 无异常 → 完成
```

### 回滚方案

每次部署都应可回滚：

```yaml
# 手动回滚工作流
name: Rollback
on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to rollback to'
        required: true

jobs:
  rollback:
    runs-on: ubuntu-latest
    steps:
      - name: Rollback deployment
        run: |
          # 部署指定的先前版本
          npx vercel rollback ${{ inputs.version }}
```

## 环境管理

```
.env.example       → 提交到仓库（开发者模板）
.env                → 不提交（本地开发）
.env.test           → 提交到仓库（测试环境，不含真实密钥）
CI secrets          → 存储在 GitHub Secrets / vault 中
Production secrets  → 存储在部署平台 / vault 中
```

CI 环境中绝不应存放生产密钥。为 CI 测试使用独立的密钥。

## CI 之外的自动化

### Dependabot / Renovate

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
    open-pull-requests-limit: 5
```

### 构建值班（Build Cop）角色

指定专人负责保持 CI 绿灯。当构建失败时，构建值班的职责是修复或回退——而不是由导致失败的人来处理。这可以防止构建持续处于失败状态，因为每个人都以为别人会去修。

### PR 检查

- **必须审查：** 合并前至少需要 1 个批准
- **必须通过状态检查：** CI 必须在合并前通过
- **分支保护：** 禁止对 main 进行 force-push
- **自动合并：** 如果所有检查通过且已批准，则自动合并

## CI 优化

当 pipeline 超过 10 分钟时，按以下影响程度依次应用这些策略：

```
CI 运行缓慢？
├── 缓存依赖
│   └── 使用 actions/cache 或 setup-node 的 cache 选项缓存 node_modules
├── 并行运行任务
│   └── 将 lint、typecheck、test、build 拆分为独立的并行 job
├── 仅运行变更相关的任务
│   └── 使用路径过滤跳过不相关的 job（例如，跳过纯文档 PR 的 e2e 测试）
├── 使用矩阵构建
│   └── 将测试套件分片到多个 runner 上
├── 优化测试套件
│   └── 将慢速测试从关键路径中移除，改为定时运行
└── 使用更大规格的 runner
    └── GitHub 托管的大规格 runner 或自托管 runner（适用于 CPU 密集型构建）
```

**示例：缓存与并行化**
```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: 'npm' }
      - run: npm ci
      - run: npm run lint

  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: 'npm' }
      - run: npm ci
      - run: npx tsc --noEmit

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: 'npm' }
      - run: npm ci
      - run: npm test -- --coverage
```

## 常见的自我合理化

| 合理化说法 | 现实 |
|---|---|
| "CI 太慢了" | 优化 pipeline（参见下方 CI 优化），而不是跳过它。5 分钟的 pipeline 能节省数小时的调试时间。 |
| "这个改动很小，跳过 CI" | 小改动同样会破坏构建。而且 CI 对小改动的运行速度本来就很快。 |
| "测试不稳定，重新跑一下就行" | 不稳定的测试会掩盖真正的 bug，浪费所有人的时间。修复不稳定性本身。 |
| "以后再添加 CI" | 没有 CI 的项目会不断积累损坏的状态。从第一天就搭建好。 |
| "手动测试足够了" | 手动测试无法扩展，也不具备可重复性。尽可能自动化。 |

## 危险信号

- 项目中没有 CI pipeline
- CI 失败被忽略或静默处理
- 为了让 pipeline 通过而在 CI 中禁用测试
- 未经预发布环境验证直接部署到生产
- 没有回滚机制
- 密钥存储在代码或 CI 配置文件中（而非密钥管理器）
- CI 运行时间很长却没有优化

## 验证

在搭建或修改 CI 后：

- [ ] 所有质量门禁均已就位（lint、类型检查、测试、构建、安全审计）
- [ ] Pipeline 在每次 PR 和 push 到 main 时运行
- [ ] 失败会阻止合并（已配置分支保护）
- [ ] CI 结果能反馈到开发循环中
- [ ] 密钥存储在密钥管理器中，而非代码中
- [ ] 部署具备回滚机制
- [ ] Pipeline 中测试套件的运行时间在 10 分钟以内
