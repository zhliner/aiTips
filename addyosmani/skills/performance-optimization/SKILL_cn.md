---
name: performance-optimization
description: 优化应用性能。当存在性能要求、怀疑出现性能回退，或 Core Web Vitals 和加载时间需要改善时使用。当性能分析发现需要修复的瓶颈时使用。
---

# 性能优化（Performance Optimization）

## 概述

在优化之前先进行度量。没有度量的性能优化就是在猜测——而猜测会导致过早优化，增加复杂性却无法改善真正重要的指标。先做性能分析，找到实际瓶颈，修复它，再次度量。只优化那些被数据证明重要的部分。

## 使用时机

- 规格说明中存在性能要求（加载时间预算、响应时间 SLA）
- 用户或监控系统报告了缓慢行为
- Core Web Vitals 分数低于阈值
- 你怀疑某次变更引入了回退
- 正在构建处理大数据集或高流量的功能

**不应使用的时机：** 在没有问题证据之前不要优化。过早优化会增加复杂性，其代价超过所获得的性能收益。

## Core Web Vitals 目标值

| 指标 | 良好 | 需要改善 | 较差 |
|--------|------|-------------------|------|
| **LCP**（Largest Contentful Paint） | ≤ 2.5s | ≤ 4.0s | > 4.0s |
| **INP**（Interaction to Next Paint） | ≤ 200ms | ≤ 500ms | > 500ms |
| **CLS**（Cumulative Layout Shift） | ≤ 0.1 | ≤ 0.25 | > 0.25 |

## 优化工作流

```
1. 度量    → 用真实数据建立基线
2. 识别    → 找到实际瓶颈（而非假设的）
3. 修复    → 解决具体的瓶颈
4. 验证    → 再次度量，确认改善
5. 防护    → 添加监控或测试以防止回退
```

### 第 1 步：度量

两种互补的方法——同时使用：

- **合成测试（Lighthouse、DevTools Performance 面板）：** 受控条件，可复现。最适合 CI 回退检测和隔离特定问题。
- **RUM（web-vitals 库、CrUX）：** 真实条件下的真实用户数据。必须用来验证修复是否真正改善了用户体验。

**前端：**
```bash
# 合成测试：Chrome DevTools 中的 Lighthouse（或 CI）
# Chrome DevTools → Performance 面板 → 录制
# Chrome DevTools MCP → Performance trace

# RUM：代码中使用 Web Vitals 库
import { onLCP, onINP, onCLS } from 'web-vitals';

onLCP(console.log);
onINP(console.log);
onCLS(console.log);
```

**后端：**
```bash
# 响应时间日志记录
# 应用性能监控（APM）
# 带计时的数据库查询日志

# 简单计时
console.time('db-query');
const result = await db.query(...);
console.timeEnd('db-query');
```

### 从哪里开始度量

根据症状决定首先度量什么：

```
什么很慢？
├── 首次页面加载
│   ├── 包体积过大？ --> 度量包体积，检查代码分割
│   ├── 服务器响应慢？ --> 在 DevTools Network 瀑布图中度量 TTFB
│   │   ├── DNS 耗时长？ --> 为已知源添加 dns-prefetch / preconnect
│   │   ├── TCP/TLS 耗时长？ --> 启用 HTTP/2，检查边缘部署，keep-alive
│   │   └── 等待（服务器）耗时长？ --> 分析后端性能，检查查询和缓存
│   └── 渲染阻塞资源？ --> 检查 Network 瀑布图中阻塞的 CSS/JS
├── 交互感觉迟钝
│   ├── 点击时 UI 冻结？ --> 分析主线程，查找长任务（>50ms）
│   ├── 表单输入延迟？ --> 检查重渲染、受控组件开销
│   └── 动画卡顿？ --> 检查布局抖动、强制回流
├── 导航后的页面
│   ├── 数据加载？ --> 度量 API 响应时间，检查是否存在瀑布流
│   └── 客户端渲染？ --> 分析组件渲染时间，检查 N+1 请求
└── 后端 / API
    ├── 单个端点慢？ --> 分析数据库查询，检查索引
    ├── 所有端点都慢？ --> 检查连接池、内存、CPU
    └── 间歇性变慢？ --> 检查锁竞争、GC 暂停、外部依赖
```

### 第 2 步：识别瓶颈

按类别划分的常见瓶颈：

**前端：**

| 症状 | 可能原因 | 调查方法 |
|---------|-------------|---------------|
| LCP 慢 | 大图、渲染阻塞资源、服务器响应慢 | 检查 Network 瀑布图、图片尺寸 |
| CLS 高 | 图片未设置尺寸、延迟加载的内容、字体偏移 | 检查布局偏移归因 |
| INP 差 | 主线程上 JavaScript 过重、大规模 DOM 更新 | 检查 Performance 追踪中的长任务 |
| 初始加载慢 | 包体积大、网络请求多 | 检查包体积、代码分割 |

**后端：**

| 症状 | 可能原因 | 调查方法 |
|---------|-------------|---------------|
| API 响应慢 | N+1 查询、缺少索引、未优化的查询 | 检查数据库查询日志 |
| 内存增长 | 引用泄漏、无界缓存、大数据载荷 | 堆快照分析 |
| CPU 飙升 | 同步重计算、正则回溯 | CPU 性能分析 |
| 高延迟 | 缺少缓存、冗余计算、网络跳转 | 在技术栈中追踪请求 |

### 第 3 步：修复常见反模式

#### N+1 查询（后端）

```typescript
// 错误：N+1 — 每个任务单独查询一次 owner
const tasks = await db.tasks.findMany();
for (const task of tasks) {
  task.owner = await db.users.findUnique({ where: { id: task.ownerId } });
}

// 正确：使用 join/include 的单次查询
const tasks = await db.tasks.findMany({
  include: { owner: true },
});
```

#### 无界数据获取

```typescript
// 错误：获取所有记录
const allTasks = await db.tasks.findMany();

// 正确：带分页限制
const tasks = await db.tasks.findMany({
  take: 20,
  skip: (page - 1) * 20,
  orderBy: { createdAt: 'desc' },
});
```

#### 缺少图片优化（前端）

```html
<!-- 错误：未设置尺寸，未做格式优化 -->
<img src="/hero.jpg" />

<!-- 正确：Hero / LCP 图片 — 艺术指导 + 分辨率切换，高优先级 -->
<!--
  两种技术结合：
  - 艺术指导（media）：每个断点使用不同的裁剪/构图
  - 分辨率切换（srcset + sizes）：根据屏幕密度提供合适大小的文件
-->
<picture>
  <!-- 移动端：竖版裁剪（8:10） -->
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
  <!-- 桌面端：横版裁剪（2:1） -->
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
    alt="Hero image description"
  />
</picture>

<!-- 正确：首屏以下图片 — 懒加载 + 异步解码 -->
<img
  src="/content.webp"
  width="800"
  height="400"
  loading="lazy"
  decoding="async"
  alt="Content image description"
/>
```

#### 不必要的重渲染（React）

```tsx
// 错误：每次渲染都创建新对象，导致子组件重渲染
function TaskList() {
  return <TaskFilters options={{ sortBy: 'date', order: 'desc' }} />;
}

// 正确：稳定的引用
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

#### 包体积过大

```typescript
// 现代打包工具（Vite、webpack 5+）会自动通过 tree-shaking 处理命名导入，
// 前提是依赖提供 ESM 格式并在 package.json 中标记 `sideEffects: false`。
// 在更改导入方式前先做性能分析 — 真正的收益来自代码分割和懒加载。

// 正确：对重量级、低频使用的功能使用动态导入
const ChartLibrary = lazy(() => import('./ChartLibrary'));

// 正确：路由级代码分割，包裹在 Suspense 中
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

设定预算并强制执行：

```
JavaScript 包体积：< 200KB gzip 压缩后（初始加载）
CSS：< 50KB gzip 压缩后
图片：每张 < 200KB（首屏图片）
字体：总计 < 100KB
API 响应时间：< 200ms（p95）
可交互时间：< 3.5s（4G 网络）
Lighthouse Performance 分数：≥ 90
```

**在 CI 中强制执行：**
```bash
# 包体积检查
npx bundlesize --config bundlesize.config.json

# Lighthouse CI
npx lhci autorun
```

## 另请参阅

详细的性能检查清单、优化命令和反模式参考，请参见 `references/performance-checklist.md`。


## 常见自我合理化

| 合理化说辞 | 现实 |
|---|---|
| "我们以后再优化" | 性能债务会复利增长。现在就修复明显的反模式，推迟微优化。 |
| "在我的机器上很快" | 你的机器不是用户的。在代表性的硬件和网络环境下做性能分析。 |
| "这个优化显而易见" | 如果你没有度量，你就不知道。先做性能分析。 |
| "用户不会注意到 100ms 的差异" | 研究表明 100ms 的延迟会影响转化率。用户比你想的更敏感。 |
| "框架会处理性能问题" | 框架能防止一些问题，但无法修复 N+1 查询或过大的包体积。 |

## 危险信号

- 没有性能分析数据支撑的优化
- 数据获取中的 N+1 查询模式
- 没有分页的列表端点
- 未设置尺寸、懒加载或响应式尺寸的图片
- 包体积在未经审查的情况下持续增长
- 生产环境中没有性能监控
- 到处使用 `React.memo` 和 `useMemo`（过度使用和不用一样糟糕）

## 验证

在任何性能相关的变更之后：

- [ ] 存在变更前后的度量数据（具体数值）
- [ ] 已识别并解决具体瓶颈
- [ ] Core Web Vitals 在"良好"阈值内
- [ ] 包体积未显著增加
- [ ] 新的数据获取代码中没有 N+1 查询
- [ ] 性能预算在 CI 中通过（如已配置）
- [ ] 现有测试仍然通过（优化没有破坏行为）
