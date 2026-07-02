# Accessibility Checklist（无障碍检查清单）

WCAG 2.1 AA 合规性快速参考。配合 `frontend-ui-engineering` skill 使用。

## 目录

- [基础检查](#essential-checks-基础检查)
- [常见 HTML 模式](#common-html-patterns-常见-html-模式)
- [测试工具](#testing-tools-测试工具)
- [快速参考：ARIA Live Regions](#quick-reference-aria-live-regions-快速参考aria-live-regions)
- [常见反模式](#common-anti-patterns-常见反模式)

## Essential Checks（基础检查）

### Keyboard Navigation（键盘导航）
- [ ] 所有可交互元素均可通过 Tab 键获得焦点
- [ ] 焦点顺序遵循视觉/逻辑顺序
- [ ] 焦点可见（被聚焦元素上有 outline/ring 样式）
- [ ] 自定义组件支持键盘操作（Enter 激活、Escape 关闭）
- [ ] 无键盘陷阱（用户始终可以从组件中 Tab 离开）
- [ ] 页面顶部有 skip-to-content 链接——至少（在键盘聚焦时）可见
- [ ] 模态框打开时锁定焦点，关闭时返还焦点

### Screen Readers（屏幕阅读器）
- [ ] 所有图片均有 `alt` 文本（或装饰性图片使用 `alt=""`）
- [ ] 所有表单输入均有关联的 label（`<label>` 或 `aria-label`）
- [ ] 按钮和链接具有描述性文本（而非 "Click here"）
- [ ] 纯图标按钮有 `aria-label`
- [ ] 页面有一个 `<h1>` 且标题层级不跳级
- [ ] 动态内容变更被播报（`aria-live` 区域）
- [ ] 表格具有带 scope 属性的 `<th>` 表头

### Visual（视觉）
- [ ] 文本对比度 ≥ 4.5:1（普通文本）或 ≥ 3:1（大号文本，18px 及以上）
- [ ] UI 组件相对背景对比度 ≥ 3:1
- [ ] 颜色不是传递信息的唯一方式
- [ ] 文本可缩放至 200% 而不破坏布局
- [ ] 无内容以每秒超过 3 次的频率闪烁

### Forms（表单）
- [ ] 每个输入框都有可见的 label
- [ ] 必填字段有明确标识（不能仅靠颜色）
- [ ] 错误信息具体且与对应字段关联
- [ ] 错误状态不仅通过颜色可见（图标、文本、边框）
- [ ] 表单提交错误有汇总且可获得焦点
- [ ] 已知字段使用 autocomplete（例如 `type="email" autocomplete="email"`）

### Content（内容）
- [ ] 声明了语言（`<html lang="en">`）
- [ ] 页面具有描述性的 `<title>`
- [ ] 链接与周围文本有区分（不能仅靠颜色）
- [ ] 移动端触控目标 ≥ 44x44px
- [ ] 有意义的空状态（而非空白页面）

## Common HTML Patterns（常见 HTML 模式）

### 按钮 vs. 链接

```html
<!-- 操作用 <button> -->
<button onClick={handleDelete}>Delete Task</button>

<!-- 导航用 <a> -->
<a href="/tasks/123">View Task</a>

<!-- 切勿使用 div/span 作为按钮 -->
<div onClick={handleDelete}>Delete</div>  <!-- 不好 -->
```

### 表单 Label

```html
<!-- 显式 label 关联 -->
<label htmlFor="email">Email address</label>
<input id="email" type="email" required />

<!-- 隐式包裹 -->
<label>
  Email address
  <input type="email" required />
</label>

<!-- 隐藏 label（首选可见 label） -->
<input type="search" aria-label="Search tasks" />
```

### ARIA Roles

```html
<!-- 导航 -->
<nav aria-label="Main navigation">...</nav>
<nav aria-label="Footer links">...</nav>

<!-- 状态消息 -->
<div role="status" aria-live="polite">Task saved</div>

<!-- 警报消息 -->
<div role="alert">Error: Title is required</div>

<!-- 模态对话框 -->
<dialog aria-modal="true" aria-labelledby="dialog-title">
  <h2 id="dialog-title">Confirm Delete</h2>
  ...
</dialog>

<!-- 加载状态 -->
<div aria-busy="true" aria-label="Loading tasks">
  <Spinner />
</div>
```

### 无障碍列表

```html
<ul role="list" aria-label="Tasks">
  <li>
    <input type="checkbox" id="task-1" aria-label="Complete: Buy groceries" />
    <label htmlFor="task-1">Buy groceries</label>
  </li>
</ul>
```

## Testing Tools（测试工具）

```bash
# 自动化审计
npx axe-core          # 可编程的无障碍测试
npx pa11y             # CLI 无障碍检查器

# 浏览器内
# Chrome DevTools → Lighthouse → Accessibility
# Chrome DevTools → Elements → Accessibility tree

# 屏幕阅读器测试
# macOS: VoiceOver (Cmd + F5)
# Windows: NVDA（免费）或 JAWS
# Linux: Orca
```

## Quick Reference: ARIA Live Regions（快速参考：ARIA Live Regions）

| 取值 | 行为 | 适用场景 |
|-------|----------|---------|
| `aria-live="polite"` | 在下一停顿时播报 | 状态更新、保存确认 |
| `aria-live="assertive"` | 立即播报 | 错误、时间敏感警报 |
| `role="status"` | 等同于 `polite` | 状态消息 |
| `role="alert"` | 等同于 `assertive` | 错误消息 |

## Common Anti-Patterns（常见反模式）

| 反模式 | 问题 | 修复方案 |
|---|---|---|
| 用 `div` 作为按钮 | 不可聚焦，无键盘支持 | 使用 `<button>` |
| 缺少 `alt` 文本 | 图片对屏幕阅读器不可见 | 添加描述性 `alt` |
| 仅靠颜色表示状态 | 色盲用户无法察觉 | 添加图标、文本或图案 |
| 自动播放媒体 | 使人迷失方向，且无法停止 | 添加控件，不要自动播放 |
| 无 ARIA 的自定义下拉 | 键盘/屏幕阅读器无法使用 | 使用原生 `<select>` 或正确的 ARIA listbox |
| 移除焦点 outline | 用户看不到自己所在位置 | 设计 outline 样式，不要移除 |
| 空链接/按钮 | "Link" 被播报，但无描述 | 添加文本或 `aria-label` |
| `tabindex > 0` | 破坏自然 Tab 顺序 | 仅使用 `tabindex="0"` 或 `-1` |
