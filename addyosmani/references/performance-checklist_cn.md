# 性能检查清单

Web 应用性能快速参考清单。可配合 `performance-optimization` skill 使用。

## 目录

- [Core Web Vitals 目标值](#core-web-vitals-目标值)
- [TTFB 诊断](#ttfb-诊断)
- [前端检查清单](#前端检查清单)
- [后端检查清单](#后端检查清单)
- [测量命令](#测量命令)
- [常见反模式](#常见反模式)

## Core Web Vitals 目标值

| 指标 | 良好 | 需改进 | 较差 |
|--------|------|------------|------|
| LCP (Largest Contentful Paint) | ≤ 2.5s | ≤ 4.0s | > 4.0s |
| INP (Interaction to Next Paint) | ≤ 200ms | ≤ 500ms | > 500ms |
| CLS (Cumulative Layout Shift) | ≤ 0.1 | ≤ 0.25 | > 0.25 |

## TTFB 诊断

当 TTFB 较慢（> 800ms）时，在 DevTools Network 瀑布图中逐项检查以下组件：

- [ ] **DNS 解析**慢 → 为已知源添加 `<link rel="dns-prefetch">` 或 `<link rel="preconnect">`
- [ ] **TCP/TLS 握手**慢 → 启用 HTTP/2，考虑边缘部署，验证 keep-alive
- [ ] **服务器处理**慢 → 对后端进行性能分析，检查慢查询，添加缓存

## 前端检查清单

### 图片
- [ ] 图片使用现代格式（WebP、AVIF）
- [ ] 图片使用响应式尺寸（`srcset` 和 `sizes`）
- [ ] 图片和 `<source>` 元素显式指定 `width` 和 `height`（防止 art direction 场景下的 CLS）
- [ ] 折线以下的图片使用 `loading="lazy"` 和 `decoding="async"`
- [ ] Hero/LCP 图片使用 `fetchpriority="high"` 且不启用懒加载

### JavaScript
- [ ] 打包体积在 200KB 以内（gzip 压缩后，初始加载）
- [ ] 使用动态 `import()` 对路由和重量级功能进行代码分割
- [ ] 启用 Tree shaking（确认依赖提供 ESM 格式并标记 `sideEffects: false`）
- [ ] `<head>` 中无阻塞 JavaScript（使用 `defer` 或 `async`）
- [ ] 重量级计算卸载到 Web Workers（如适用）
- [ ] 对使用相同 props 重新渲染的高开销组件使用 `React.memo()`
- [ ] 仅在性能分析显示有益时使用 `useMemo()` / `useCallback()`
- [ ] 拆分长任务（> 50ms）以保持主线程可用——这是优化 INP 的主要手段
- [ ] 在长时间运行的循环中使用 `yieldToMain` 模式，使输入事件可在代码块之间执行
- [ ] 在可用时使用现代调度 API：`scheduler.yield()`（首选）、带优先级的 `scheduler.postTask()`、仅在需要时让出的 `isInputPending()`
- [ ] 使用 `requestIdleCallback` 处理可延迟的非紧急工作（分析数据刷新、预取、预热）
- [ ] 将非关键工作从事件处理器中延迟执行（如分析、日志），以免延迟交互响应
- [ ] 第三方脚本使用 `async` / `defer` 加载，审计体积，重量级脚本（聊天组件、嵌入内容）使用 facade 模式

### CSS
- [ ] 关键 CSS 内联或预加载
- [ ] 非关键样式无渲染阻塞 CSS
- [ ] 生产环境中无 CSS-in-JS 运行时开销（使用提取方案）

### 字体
- [ ] 限制为 2–3 个字体系列，每个 2–3 个字重（每个额外字重都是额外请求）
- [ ] 仅使用 WOFF2 格式（体积最小，通用支持——跳过 WOFF/TTF/EOT）
- [ ] 尽可能自托管（第三方字体 CDN 会增加 DNS + TCP + TLS 往返）
- [ ] 预加载 LCP 关键字体：`<link rel="preload" as="font" type="font/woff2" crossorigin>`
- [ ] 使用 `font-display: swap`（或非关键字体使用 `optional`）以避免 FOIT 阻塞渲染
- [ ] 通过 `unicode-range` 进行子集化，仅加载每个页面所需的字符
- [ ] 需要多个字重/样式时考虑使用可变字体（一个文件替代多个文件）
- [ ] 使用 `size-adjust`、`ascent-override`、`descent-override` 调整回退字体指标，减少字体切换时的 CLS
- [ ] 在任何自定义字体之前考虑系统字体栈

### 网络
- [ ] 静态资源使用长 `max-age` + 内容哈希缓存
- [ ] 适当缓存 API 响应（`Cache-Control`）
- [ ] 启用 HTTP/2 或 HTTP/3
- [ ] 为已知源预连接资源（`<link rel="preconnect">`）
- [ ] 对关键非图片资源使用 `fetchpriority`（如关键 `<link rel="preload">`、折线以上 `<script>`）——不仅限于 `<img>`
- [ ] 无不必要的重定向

### 渲染
- [ ] 无布局抖动（强制同步布局）
- [ ] 动画使用 `transform` 和 `opacity`（GPU 加速）
- [ ] 长列表使用虚拟化（如 `react-window`）
- [ ] 无不必要的全页重新渲染
- [ ] 屏外区域使用 `content-visibility: auto` 配合 `contain-intrinsic-size`，跳过不可见区域的布局/绘制
- [ ] 无 `unload` 事件处理器，HTML 响应无 `Cache-Control: no-store`——保持 back/forward cache（bfcache）资格

## 后端检查清单

### 数据库
- [ ] 无 N+1 查询模式（使用预加载 / join）
- [ ] 查询具有适当的索引
- [ ] 列表接口分页（永远不要 `SELECT * FROM table`）
- [ ] 配置连接池
- [ ] 启用慢查询日志

### API
- [ ] 响应时间 < 200ms（p95）
- [ ] 请求处理器中无同步重量级计算
- [ ] 使用批量操作替代循环单独调用
- [ ] 响应压缩（gzip/brotli）
- [ ] 适当缓存（内存、Redis、CDN）

### 基础设施
- [ ] 静态资源使用 CDN
- [ ] 服务器靠近用户（或边缘部署）
- [ ] 配置水平扩展（如需要）
- [ ] 负载均衡器健康检查端点

## 测量命令

### INP 现场数据与 DevTools 工作流

1. **优先查看现场数据** — 在优化前，通过 [CrUX Vis](https://developer.chrome.com/docs/crux/vis) 或 RUM 工具查看真实用户的 INP
2. **识别慢交互** — 打开 DevTools → Performance 面板 → 交互时录制；查找点击/按键触发的长任务
3. **在中端 Android 设备上测试** — INP 问题通常只在较慢硬件上显现；使用真机或 DevTools CPU 节流（4×–6× 减速）

```bash
# Lighthouse CLI
npx lighthouse https://localhost:3000 --output json --output-path ./report.json

# 打包分析
npx webpack-bundle-analyzer stats.json
# 或 Vite：
npx vite-bundle-visualizer

# 检查打包体积
npx bundlesize

# 在代码中测量 Web Vitals
import { onLCP, onINP, onCLS } from 'web-vitals';
onLCP(console.log);
onINP(console.log);
onCLS(console.log);

# 带交互级别详情的 INP（attribution 构建）
import { onINP } from 'web-vitals/attribution';
onINP(({ value, attribution }) => {
  const { interactionTarget, inputDelay, processingDuration, presentationDelay } = attribution;
  console.log({ value, interactionTarget, inputDelay, processingDuration, presentationDelay });
});
```

## 常见反模式

| 反模式 | 影响 | 修复方案 |
|---|---|---|
| N+1 查询 | 数据库负载线性增长 | 使用 join、includes 或批量加载 |
| 无界查询 | 内存耗尽、超时 | 始终分页，添加 LIMIT |
| 缺少索引 | 数据增长时读取变慢 | 为过滤/排序列添加索引 |
| 布局抖动 | 卡顿、掉帧 | 批量 DOM 读取，然后批量写入 |
| 未优化图片 | LCP 慢、浪费带宽 | 使用 WebP、响应式尺寸、懒加载 |
| 大型打包文件 | 可交互时间慢 | 代码分割、tree shake、审计依赖 |
| 阻塞主线程 | INP 差、UI 无响应 | 使用 `scheduler.yield()` / `yieldToMain` 拆分长任务，卸载到 Web Workers |
| 内存泄漏 | 内存增长、最终崩溃 | 清理事件监听器、定时器、引用 |
