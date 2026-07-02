---
name: ci-cd-and-automation
description: 自动化 CI/CD 流水线设置。在设置或修改构建和部署流水线时使用。在需要自动化质量门禁、在 CI 中配置测试运行器或建立部署策略时使用。
---

# CI/CD 与自动化

## 概述

自动化质量门禁，确保任何变更在通过测试、lint、类型检查和构建之前无法到达生产环境。CI/CD 是其他所有技能的强制执行机制——它捕获人类和 agent 遗漏的问题，并且对每一次变更都始终如一地执行。

**左移：** 在流水线中尽早捕获问题。在 linting 阶段捕获的 bug 成本仅几分钟；同样的 bug 在生产环境中捕获则要数小时。将检查向上游移动——先静态分析，再测试，测试通过后进预发布环境，预发布通过后再进生产环境。

**越快越安全：** 更小的批次和更高频的发布降低而非增加风险。一次部署 3 个变更比部署 30 个更容易调试。高频发布本身就建立了对发布流程的信心。

## 适用场景

- 设置新项目的 CI 流水线
- 添加或修改自动化检查
- 配置部署流水线
- 确定何时变更应触发自动化验证
- 调试 CI 故障

## 质量门禁流水线

每个变更在合并前要通过以下门禁：

```
Pull Request 已创建
    │
    ▼
┌─────────────────┐
│   代码检查         │  eslint, prettier
│   ↓ 通过          │
│   类型检查         │  tsc --noEmit
│   ↓ 通过          │
│   单元测试         │  jest/vitest
│   ↓ 通过          │
│   构建             │  npm run build
│   ↓ 通过          │
│   集成测试         │  API/DB 测试
│   ↓ 通过          │
│   E2E（可选）      │  Playwright/Cypress
│   ↓ 通过          │
│   安全审计         │  npm audit
│   ↓ 通过          │
│   包体积检查       │  bundlesize 检查
└─────────────────┘
    │
    ▼
  准备好评审
```

**任何门禁都不能跳过。** 如果 lint 失败，修复代码——不要禁用规则。如果测试失败，修复代码——不要跳过测试。

## GitHub Actions 配置

### 基础 CI 流水线

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

### 带数据库集成测试

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

> **注意：** 即使是仅用于 CI 的测试数据库，也应使用 GitHub Secrets 存储凭据，而不是硬编码值。这能养成好习惯，并防止测试凭据在其他上下文中被意外复用。

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

## 将 CI 故障反馈给 Agent

CI 与 AI agent 结合的力量在于反馈循环。当 CI 失败时：

```
CI 失败
    │
    ▼
复制失败输出
    │
    ▼
反馈给 agent：
"CI 流水线失败，报错如下：
[粘贴具体错误]
请修复问题并在推送前本地验证。"
    │
    ▼
Agent 修复 → 推送 → CI 重新运行
```

**关键模式：**

```
Lint 失败 → Agent 运行 `npm run lint --fix` 并提交
类型错误  → Agent 读取错误位置并修复类型
测试失败  → Agent 遵循调试与错误恢复技能
构建错误  → Agent 检查配置和依赖
```

## 部署策略

### 预览部署

每个 PR 获得一个用于手动测试的预览部署：

```yaml
# 在 PR 时部署预览 (Vercel/Netlify 等)
deploy-preview:
  runs-on: ubuntu-latest
  if: github.event_name == 'pull_request'
  steps:
    - uses: actions/checkout@v4
    - name: Deploy preview
      run: npx vercel --token=${{ secrets.VERCEL_TOKEN }}
```

### 功能开关（Feature Flags）

功能开关将部署与发布解耦。将不完整或有风险的功能部署在开关后面，这样你可以：

- **先部署代码，不启用。** 尽早合并到 main，准备好后再启用。
- **不重新部署即可回滚。** 禁用开关而非回退代码。
- **灰度新功能。** 先对 1% 用户启用，再到 10%，再到 100%。
- **运行 A/B 测试。** 比较有和没有该功能的行为差异。

```typescript
// 简单的功能开关模式
if (featureFlags.isEnabled('new-checkout-flow', { userId })) {
  return renderNewCheckout();
}
return renderLegacyCheckout();
```

**开关生命周期：** 创建 → 启用测试 → 灰度 → 全量推出 → 移除开关和相关死代码。永远存在的开关会变成技术债务——创建时设定清理日期。

### 分阶段上线

```
PR 合并到 main
    │
    ▼
  预发布部署（自动）
    │ 手动验证
    ▼
  生产部署（手动触发或预发布后自动）
    │
    ▼
  监控错误（15 分钟窗口期）
    │
    ├── 检测到错误 → 回滚
    └── 无问题 → 完成
```

### 回滚计划

每次部署都应该是可逆的：

```yaml
# 手动回滚工作流
name: Rollback
on:
  workflow_dispatch:
    inputs:
      version:
        description: '要回滚到的版本'
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
.env.example       → 已提交（开发人员模板）
.env                → 不提交（本地开发）
.env.test           → 已提交（测试环境，无真实密钥）
CI secrets          → 存储在 GitHub Secrets / vault 中
Production secrets  → 存储在部署平台 / vault 中
```

CI 永远不应拥有生产环境密钥。为 CI 测试使用单独的密钥。

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

### Build Cop 角色

指定专人负责保持 CI 绿色。当构建中断时，Build Cop 的职责是修复或回退——而非那位导致中断的开发者。这能防止构建失败不断累积，而每个人都以为别人会修复它。

### PR 检查

- **必需评审：** 合并前至少 1 次审批
- **必需状态检查：** CI 必须在合并前通过
- **分支保护：** 禁止 force-push 到 main
- **自动合并：** 当所有检查通过且获得审批时，自动合并

## CI 优化

当流水线超过 10 分钟时，按影响程度依次应用以下策略：

```
CI 流水线太慢？
├── 缓存依赖
│   └── 对 node_modules 使用 actions/cache 或 setup-node cache 选项
├── 并行运行任务
│   └── 将 lint、typecheck、test、build 拆分到独立的并行任务中
├── 只运行变更相关的内容
│   └── 使用路径筛选跳过无关任务（如仅为文档变更的 PR 跳过 e2e）
├── 使用矩阵构建
│   └── 将测试套件分片到多个运行器上
├── 优化测试套件
│   └── 将慢速测试移出关键路径，改为按计划运行
└── 使用更大的运行器
    └── GitHub 托管的较大运行器或自托管运行器用于 CPU 密集构建
```

**示例：缓存与并行**
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

## 常见合理化借口

| 借口 | 现实 |
|---|---|
| "CI 太慢了" | 优化流水线（参见下方 CI 优化），而不是跳过它。一个 5 分钟的流水线能防止数小时的调试。 |
| "这个改动很琐碎，跳过 CI 吧" | 琐碎的改动也能破坏构建。CI 对于琐碎改动本来也很快。 |
| "这个测试不稳定，重新跑就好" | 不稳定的测试会掩盖真正的 bug，浪费每个人的时间。修复不稳定本身。 |
| "我们后面再加 CI" | 没有 CI 的项目会累积各种破损状态。从第一天就设置好。 |
| "手动测试就够了" | 手动测试不可扩展，也无法重复。自动化你能自动化的。 |

## 危险信号

- 项目中没有 CI 流水线
- CI 失败被忽略或静默处理
- 为了让流水线通过而禁用测试
- 未经预发布验证就直接部署到生产
- 没有回滚机制
- 密钥存储在代码或 CI 配置文件中（而非 secrets manager）
- CI 运行时间长但未做任何优化

## 验证

设置或修改 CI 之后：

- [ ] 所有质量门禁均已到位（lint、类型、测试、构建、审计）
- [ ] 每个 PR 和 push 到 main 时都运行流水线
- [ ] 失败会阻止合并（已配置分支保护）
- [ ] CI 结果反馈到开发循环中
- [ ] 密钥存储在 secrets manager 中，而非代码中
- [ ] 部署有回滚机制
- [ ] 测试套件的流水线在 10 分钟以内运行完成
