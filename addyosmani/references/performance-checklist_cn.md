# Performance Checklist（性能检查清单）

Web 应用性能快速参考。配合 `performance-optimization` skill 使用。

## 目录

- [Core Web Vitals Targets（Core Web Vitals 目标值）](#core-web-vitals-targetscore-web-vitals-目标值)
- [TTFB Diagnosis（TTFB 诊断）](#ttfb-diagnosisttfb-诊断)
- [Frontend Checklist（前端检查清单）](#frontend-checklist前端检查清单)
- [Backend Checklist（后端检查清单）](#backend-checklist后端检查清单)
- [Measurement Commands（测量命令）](#measurement-commands测量命令)
- [Common Anti-Patterns（常见反模式）](#common-anti-patterns常见反模式)

## Core Web Vitals Targets（Core Web Vitals 目标值）

| 指标 | 良好 | 需要改进 | 差 |
|--------|------|------------|------|
| LCP（Largest Contentful Paint，最大内容绘制） | ≤ 2.5s | ≤ 4.0s | > 4.0s |
| INP（Interaction to Next Paint，交互到下次绘制） | ≤ 200ms | ≤ 500ms | > 500ms |
| CLS（Cumulative Layout Shift，累积布局偏移） | ≤ 0.1 | ≤ 0.25 | > 0.25 |

## TTFB Diagnosis（TTFB 诊断）

当 TTFB 过慢（> 800ms）时，在 DevTools Network 瀑布图中检查每个环节：

- [ ] **DNS 解析**慢 → 为已知源添加 `<link rel="dns-prefetch">` 或 `<link rel="preconnect">`
- [ ] **TCP/TLS 握手**慢 → 启用 HTTP/2，考虑边缘部署，验证 keep-alive
- [ ] **服务器处理**慢 → 分析后端，检查慢查询，添加缓存

## Frontend Checklist（前端检查清单）

### Images（图片）
- [ ] 图片使用现代格式（WebP、AVIF）
- [ ] 图片响应式适配尺寸（`srcset` 和 `sizes`）
- [ ] 图片和 `<source>` 元素具有显式的 `width` 和 `height`（防止 art direction 中的 CLS）
- [ ] 首屏以下的图片使用 `loading="lazy"` 和 `decoding="async"`
- [ ] Hero/LCP 图片使用 `fetchpriority="high"` 且不使用懒加载

### JavaScript
- [ ] Bundle 大小在 200KB gzip 以下（初始加载）
- [ ] 路由和重型功能使用动态 `import()` 进行代码分割
- [ ] Tree shaking 已启用（验证依赖是否提供 ESM 并标记 `sideEffects: false`）
- [ ] `<head>` 中无阻塞 JavaScript（使用 `defer` 或 `async`）
- [ ] 重型计算卸载到 Web Worker（如适用）
- [ ] 对相同 props 重新渲染的昂贵组件使用 `React.memo()`
- [ ] 仅在 profiling 显示收益时使用 `useMemo()` / `useCallback()`
- [ ] 长任务（> 50ms）被拆解以保持主线程可用——INP 的主要优化方向
- [ ] 在长时间运行的循环内使用 `yieldToMain` 模式，使各片段之间能处理输入事件
- [ ] 在可用时使用现代调度 API：`scheduler.yield()`（首选）、带优先级的 `scheduler.postTask()`、仅在需要时才让出主线程的 `isInputPending()`
- [ ] 对可延迟的非紧急工作使用 `requestIdleCallback`（分析数据上报、预取、预热）
- [ ] 非关键工作从事件处理器中延迟执行（如分析、日志），以免延迟对交互的响应
- [ ] 第三方脚本使用 `async` / `defer` 加载，审计其大小，重型组件（聊天组件、嵌入组件）用 facade 替代

### CSS
- [ ] 关键 CSS 已内联或预加载
- [ ] 非关键样式无不阻塞渲染的 CSS
- [ ] 生产环境中无 CSS-in-JS 运行时开销（使用提取方式）

### Fonts（字体）
- [ ] 限制为 2–3 个字体系列，每个 2–3 种字重（每种额外字重都是一次额外请求）
- [ ] 仅使用 WOFF2 格式（体积最小，普遍支持——跳过 WOFF/TTF/EOT）
- [ ] 尽可能自托管（第三方字体 CDN 会增加 DNS + TCP + TLS 往返）
- [ ] LCP 关键字体已预加载：`<link rel="preload" as="font" type="font/woff2" crossorigin>`
- [ ] `font-display: swap`（或对非关键字体使用 `optional`）以避免 FOIT 阻塞渲染
- [ ] 通过 `unicode-range` 子集化，仅提供每个页面所需的字形
- [ ] 当需要多种字重/样式时考虑 Variable Font（一个文件替代多个文件）
- [ ] 使用 `size-adjust`、`ascent-override`、`descent-override` 调整备选字体指标，以减少字体切换时的 CLS
- [ ] 在使用任何自定义字体之前先考虑系统字体栈

### Network（网络）
- [ ] 静态资源使用长 `max-age` + 内容哈希进行缓存
- [ ] API 响应在合适处使用缓存（`Cache-Control`）
- [ ] HTTP/2 或 HTTP/3 已启用
- [ ] 对已知源使用资源预连接（`<link rel="preconnect">`）
- [ ] 对关键非图片资源使用 `fetchpriority`（例如关键 `<link rel="preload">`、首屏 `<script>`）——不仅是 `<img>`
- [ ] 无不必要的重定向

### Rendering（渲染）
- [ ] 无 layout thrashing（强制同步布局）
- [ ] 动画使用 `transform` 和 `opacity`（GPU 加速）
- [ ] 长列表使用虚拟化（如 `react-window`）
- [ ] 无不必要的整页重新渲染
- [ ] 屏幕外区域使用 `content-visibility: auto` 配合 `contain-intrinsic-size` 跳过不可见区域的布局/绘制
- [ ] 无 `unload` 事件处理器，且 HTML 响应上无 `Cache-Control: no-store`——保持 back/forward cache（bfcache）可用性

## Backend Checklist（后端检查清单）

### Database（数据库）
- [ ] 无 N+1 查询模式（使用 eager loading / joins）
- [ ] 查询有适当的索引
- [ ] 列表端点已分页（绝不允许 `SELECT * FROM table`）
- [ ] 连接池已配置
- [ ] 慢查询日志已启用

### API
- [ ] 响应时间 < 200ms（p95）
- [ ] 请求处理器中无同步重型计算
- [ ] 使用批量操作而非单次调用的循环
- [ ] 响应已压缩（gzip/brotli）
- [ ] 适当的缓存（内存缓存、Redis、CDN）

### Infrastructure（基础设施）
- [ ] 静态资源使用 CDN
- [ ] 服务器离用户较近（或边缘部署）
- [ ] 水平扩展已配置（如需要）
- [ ] 负载均衡器的健康检查端点

## Measurement Commands（测量命令）

### INP 现场数据和 DevTools 工作流

1. **先看现场数据**——在优化前，通过 [CrUX Vis](https://developer.chrome.com/docs/crux/vis) 或你的 RUM 工具检查真实用户的 INP
2. **识别慢交互**——打开 DevTools → Performance 面板 → 录制时进行交互操作；查找由点击/按键触发的长任务
3. **在中等性能的 Android 设备上测试**——INP 问题通常只在较慢的硬件上出现；使用真实设备或 DevTools CPU 节流（4×–6× 减速）

```bash
# Lighthouse CLI
npx lighthouse https://localhost:3000 --output json --output-path ./report.json

# Bundle 分析
npx webpack-bundle-analyzer stats.json
# 或 Vite：
npx vite-bundle-visualizer

# 检查 Bundle 大小
npx bundlesize

# 代码中的 Web Vitals
import { onLCP, onINP, onCLS } from 'web-vitals';
onLCP(console.log);
onINP(console.log);
onCLS(console.log);

# 带交互级别详情的 INP（attribution build）
import { onINP } from 'web-vitals/attribution';
onINP(({ value, attribution }) => {
  const { interactionTarget, inputDelay, processingDuration, presentationDelay } = attribution;
  console.log({ value, interactionTarget, inputDelay, processingDuration, presentationDelay });
});
```

## Common Anti-Patterns（常见反模式）

| 反模式 | 影响 | 修复方案 |
|---|---|---|
| N+1 查询 | 数据库负载线性增长 | 使用 join、include 或批量加载 |
| 无限制查询 | 内存耗尽、超时 | 始终分页，添加 LIMIT |
| 缺少索引 | 数据增长时读取变慢 | 为过滤/排序列添加索引 |
| Layout thrashing | 卡顿、丢帧 | 批量 DOM 读取，再批量写入 |
| 未优化的图片 | LCP 慢、浪费带宽 | 使用 WebP、响应式尺寸、懒加载 |
| 大体积 Bundle | Time to Interactive 慢 | 代码分割、tree shake、审计依赖 |
| 阻塞主线程 | INP 差、UI 无响应 | 使用 `scheduler.yield()` / `yieldToMain` 拆分长任务，卸载到 Web Worker |
| 内存泄漏 | 内存持续增长，最终崩溃 | 清理 listener、interval、ref |
