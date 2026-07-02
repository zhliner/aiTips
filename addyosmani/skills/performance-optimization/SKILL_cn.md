---
name: performance-optimization
description: 优化应用性能。当存在性能要求、怀疑性能回归、或需要改善 Core Web Vitals 或加载时间时使用。当分析发现瓶颈需要修复时使用。
---

# 性能优化

## 概述

先测量，再优化。没有测量的性能工作是猜测——猜测会导致过早优化，增加复杂性而不改善真正重要的东西。先分析，找到真正的瓶颈，修复它，再次测量。仅优化被测量证明有价值的部分。

## 何时使用

- Spec 中存在性能要求（加载时间预算、响应时间 SLA）
- 用户或监控报告运行缓慢
- Core Web Vitals 得分低于阈值
- 你怀疑某个变更引入了性能回归
- 构建处理大数据集或高流量的功能

**何时不使用：** 在有证据证明存在问题之前不要优化。过早优化增加的复杂性，其成本超过其带来的性能收益。

## Core Web Vitals 目标

| 指标 | 良好 | 需要改进 | 较差 |
|--------|------|-------------------|------|
| **LCP**（最大内容绘制） | ≤ 2.5s | ≤ 4.0s | > 4.0s |
| **INP**（下一次绘制交互） | ≤ 200ms | ≤ 500ms | > 500ms |
| **CLS**（累积布局偏移） | ≤ 0.1 | ≤ 0.25 | > 0.25 |

## 优化工作流

```
1. MEASURE（测量）  → 用真实数据建立基线
2. IDENTIFY（识别） → 找到真正的瓶颈（而非假想的）
3. FIX（修复）      → 处理具体的瓶颈
4. VERIFY（验证）   → 再次测量，确认改善
5. GUARD（防护）    → 添加监控或测试以防止回归
```

### 第 1 步：测量

两种互补的方法——两者都要用：

- **合成测试（Lighthouse、DevTools Performance 面板）：** 受控条件，可复现。最适合 CI 中的回归检测和隔离特定问题。
- **真实用户监控 / RUM（web-vitals 库、CrUX）：** 在真实条件下的真实用户数据。验证一个修复是否真正改善了用户体验所必需的手段。

**前端：**
```bash
# 合成测试: Chrome DevTools 中的 Lighthouse（或 CI 中）
# Chrome DevTools → Performance 面板 → Record
# Chrome DevTools MCP → Performance trace

# RUM: 代码中的 Web Vitals 库
import { onLCP, onINP, onCLS } from 'web-vitals';

onLCP(console.log);
onINP(console.log);
onCLS(console.log);
```

**后端：**
```bash
# 响应时间日志
# 应用性能监控 (APM)
# 数据库查询日志，附带耗时

# 简单计时
console.time('db-query');
const result = await db.query(...);
console.timeEnd('db-query');
```

### 从何开始测量

根据症状决定首先测量什么：

```
哪里慢？
├── 首次页面加载
│   ├── Bundle 太大？--> 测量 bundle 大小，检查代码分割
│   ├── 服务器响应慢？--> 在 DevTools Network 瀑布图中测量 TTFB
│   │   ├── DNS 时间长？--> 为已知来源添加 dns-prefetch / preconnect
│   │   ├── TCP/TLS 时间长？--> 启用 HTTP/2，检查边缘部署、keep-alive
│   │   └── Waiting (服务器) 时间长？--> 分析后端，检查查询和缓存
│   └── 渲染阻塞资源？--> 检查网络瀑布图中阻塞的 CSS/JS
├── 交互感觉迟钝
│   ├── 点击时 UI 卡死？--> 分析主线程，查找长任务 (>50ms)
│   ├── 表单输入延迟？--> 检查重渲染、受控组件开销
│   └── 动画卡顿？--> 检查布局抖动、强制回流
├── 页面导航后
│   ├── 数据加载慢？--> 测量 API 响应时间，检查是否存在请求瀑布
│   └── 客户端渲染慢？--> 分析组件渲染时间，检查 N+1 请求
└── 后端 / API
    ├── 单个端点慢？--> 分析数据库查询，检查索引
    ├── 所有端点都慢？--> 检查连接池、内存、CPU
    └── 间歇性慢？--> 检查锁竞争、GC 暂停、外部依赖
```

### 第 2 步：识别瓶颈

按类别列出的常见瓶颈：

**前端：**

| 症状 | 可能原因 | 调查方向 |
|---------|-------------|---------------|
| LCP 慢 | 大图片、渲染阻塞资源、服务器慢 | 检查网络瀑布图、图片大小 |
| CLS 高 | 无尺寸的图片、延迟加载的内容、字体偏移 | 检查布局偏移归因 |
| INP 差 | 主线程上的繁重 JavaScript、大块 DOM 更新 | 检查 Performance trace 中的长任务 |
| 初始加载慢 | 大 bundle、过多网络请求 | 检查 bundle 大小、代码分割 |

**后端：**

| 症状 | 可能原因 | 调查方向 |
|---------|-------------|---------------|
| API 响应慢 | N+1 查询、缺失索引、未优化的查询 | 检查数据库查询日志 |
| 内存增长 | 泄漏的引用、无界缓存、大载荷 | 堆快照分析 |
| CPU 尖峰 | 同步繁重计算、正则表达式回溯 | CPU 分析 |
| 高延迟 | 缺少缓存、冗余计算、网络跳数 | 跟踪请求在栈中的路径 |

### 第 3 步：修复常见反模式

#### N+1 查询（后端）

```typescript
// BAD: N+1 — 每个任务一条查询来获取所有者
const tasks = await db.tasks.findMany();
for (const task of tasks) {
  task.owner = await db.users.findUnique({ where: { id: task.ownerId } });
}

// GOOD: 带 join/include 的单次查询
const tasks = await db.tasks.findMany({
  include: { owner: true },
});
```

#### 无界数据获取

```typescript
// BAD: 获取所有记录
const allTasks = await db.tasks.findMany();

// GOOD: 带限制的分页
const tasks = await db.tasks.findMany({
  take: 20,
  skip: (page - 1) * 20,
  orderBy: { createdAt: 'desc' },
});
```

#### 缺少图片优化（前端）

```html
<!-- BAD: 无尺寸，无格式优化 -->
<img src="/hero.jpg" />

<!-- GOOD: Hero / LCP 图片 — 美术指导 + 分辨率切换，高优先级 -->
<!--
  两种技术结合使用：
  - 美术指导 (media): 每个断点不同的裁剪/构图
  - 分辨率切换 (srcset + sizes): 每个屏幕密度下的合适文件大小
-->
<picture>
  <!-- 移动端: 竖向裁剪 (8:10) -->
  <source
    media="(max-width: 767px)"
    srcset="/hero-mobile-400.avif 400w, /hero-mobile-800.avif 800w"
    sizes="100vw"
    width="800"
    height="1000"
    type="image/avif"
  />
  <source
    media="(max-width: 767px)"
    srcset="/hero-mobile-400.webp 400w, /hero-mobile-800.webp 800w"
    sizes="100vw"
    width="800"
    height="1000"
    type="image/webp"
  />
  <!-- 桌面端: 横向裁剪 (2:1) -->
  <source
    srcset="/hero-800.avif 800w, /hero-1200.avif 1200w, /hero-1600.avif 1600w"
    sizes="(max-width: 1200px) 100vw, 1200px"
    width="1200"
    height="600"
    type="image/avif"
  />
  <source
    srcset="/hero-800.webp 800w, /hero-1200.webp 1200w, /hero-1600.webp 1600w"
    sizes="(max-width: 1200px) 100vw, 1200px"
    width="1200"
    height="600"
    type="image/webp"
  />
  <img
    src="/hero-desktop.jpg"
    width="1200"
    height="600"
    fetchpriority="high"
    alt="Hero 图片描述"
  />
</picture>

<!-- GOOD: 首屏以下的图片 — 懒加载 + 异步解码 -->
<img
  src="/content.webp"
  width="800"
  height="400"
  loading="lazy"
  decoding="async"
  alt="内容图片描述"
/>
```

#### 不必要的重渲染 (React)

```tsx
// BAD: 每次渲染都创建新对象，导致子组件重渲染
function TaskList() {
  return <TaskFilters options={{ sortBy: 'date', order: 'desc' }} />;
}

// GOOD: 稳定的引用
const DEFAULT_OPTIONS = { sortBy: 'date', order: 'desc' } as const;
function TaskList() {
  return <TaskFilters options={DEFAULT_OPTIONS} />;
}

// 对开销大的组件使用 React.memo
const TaskItem = React.memo(function TaskItem({ task }: Props) {
  return <div>{/* 开销大的渲染 */}</div>;
});

// 对开销大的计算使用 useMemo
function TaskStats({ tasks }: Props) {
  const stats = useMemo(() => calculateStats(tasks), [tasks]);
  return <div>{stats.completed} / {stats.total}</div>;
}
```

#### Bundle 过大

```typescript
// 现代打包工具 (Vite、webpack 5+) 会自动通过 tree-shaking 处理具名导入，
// 前提是依赖提供 ESM 格式且在 package.json 中标记为 `sideEffects: false`。
// 在更改导入风格之前先分析——真正的收益来自拆包和懒加载。

// GOOD: 对繁重且不常用的功能使用动态导入
const ChartLibrary = lazy(() => import('./ChartLibrary'));

// GOOD: 路由级代码分割，包裹在 Suspense 中
const SettingsPage = lazy(() => import('./pages/Settings'));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <SettingsPage />
    </Suspense>
  );
}
```

#### 缺少缓存（后端）

```typescript
// 缓存频繁读取、很少变化的数据
const CACHE_TTL = 5 * 60 * 1000; // 5 分钟
let cachedConfig: AppConfig | null = null;
let cacheExpiry = 0;

async function getAppConfig(): Promise<AppConfig> {
  if (cachedConfig && Date.now() < cacheExpiry) {
    return cachedConfig;
  }
  cachedConfig = await db.config.findFirst();
  cacheExpiry = Date.now() + CACHE_TTL;
  return cachedConfig;
}

// 静态资源的 HTTP 缓存头
app.use('/static', express.static('public', {
  maxAge: '1y',           // 缓存 1 年
  immutable: true,        // 永不重新验证（在文件名中使用内容哈希）
}));

// API 响应的 Cache-Control
res.set('Cache-Control', 'public, max-age=300'); // 5 分钟
```

## 性能预算

设置预算并强制执行：

```
JavaScript bundle: < 200KB gzip 后（初始加载）
CSS: < 50KB gzip 后
图片: < 200KB 每张（首屏以上）
字体: < 100KB 总计
API 响应时间: < 200ms (p95)
Time to Interactive: < 3.5s 在 4G 网络下
Lighthouse Performance 分数: ≥ 90
```

**在 CI 中强制执行：**
```bash
# Bundle 大小检查
npx bundlesize --config bundlesize.config.json

# Lighthouse CI
npx lhci autorun
```

## 另请参阅

关于详细的性能检查清单、优化命令和反模式参考，参见 `references/performance-checklist.md`。

## 常见借口

| 借口 | 现实 |
|---|---|
| "我们后面再优化" | 性能债务会复利增长。现在修复明显的反模式，推迟微优化。 |
| "在我机器上很快" | 你的机器不是用户的机器。在代表性硬件和网络条件下分析。 |
| "这个优化是显而易见的" | 如果你没有测量，你就不知道。先分析。 |
| "用户不会注意到 100ms" | 研究表明 100ms 的延迟会影响转化率。用户比你想象的更敏感。 |
| "框架会处理性能问题" | 框架能预防一些问题，但无法修复 N+1 查询或过大的 bundle。 |

## 警示信号

- 没有分析数据支持的优化
- 数据获取中的 N+1 查询模式
- 列表端点没有分页
- 图片没有尺寸、懒加载或响应式尺寸
- Bundle 大小在没有审查的情况下增长
- 生产环境中没有性能监控
- 到处使用 `React.memo` 和 `useMemo`（过度使用和不够用一样糟糕）

## 验证

在任何与性能相关的变更之后：

- [ ] 存在变更前后的测量数据（具体数字）
- [ ] 具体的瓶颈已被识别和处理
- [ ] Core Web Vitals 在"良好"阈值内
- [ ] Bundle 大小没有显著增加
- [ ] 新增的数据获取代码中没有 N+1 查询
- [ ] 性能预算在 CI 中通过（如果已配置）
- [ ] 现有测试仍然通过（优化没有破坏行为）
