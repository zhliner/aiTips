# 无障碍检查清单

WCAG 2.1 AA 合规快速参考。配合 `frontend-ui-engineering` skill 使用。

## 目录

- [必要检查项](#必要检查项)
- [常见 HTML 模式](#常见-html-模式)
- [测试工具](#测试工具)
- [快速参考：ARIA Live Regions](#快速参考aria-live-regions)
- [常见反模式](#常见反模式)

## 必要检查项

### 键盘导航
- [ ] 所有交互元素可通过 Tab 键聚焦
- [ ] 聚焦顺序与视觉/逻辑顺序一致
- [ ] 焦点可见（聚焦元素具有 outline/ring）
- [ ] 自定义组件支持键盘操作（Enter 激活，Escape 关闭）
- [ ] 无键盘陷阱（用户始终可以 Tab 离开组件）
- [ ] 页面顶部有 Skip-to-content 链接——至少在键盘聚焦时可见
- [ ] 模态框打开时锁定焦点，关闭时焦点返回

### 屏幕阅读器
- [ ] 所有图片具有 `alt` 文本（装饰性图片使用 `alt=""`）
- [ ] 所有表单输入具有关联标签（`<label>` 或 `aria-label`）
- [ ] 按钮和链接具有描述性文本（而非"点击此处"）
- [ ] 仅含图标的按钮具有 `aria-label`
- [ ] 页面有且仅有一个 `<h1>`，标题层级不跳级
- [ ] 动态内容变化被播报（`aria-live` 区域）
- [ ] 表格具有带 scope 属性的 `<th>` 表头

### 视觉
- [ ] 文本对比度 ≥ 4.5:1（普通文本）或 ≥ 3:1（大号文本，18px+）
- [ ] UI 组件与背景对比度 ≥ 3:1
- [ ] 颜色不是传达信息的唯一方式
- [ ] 文本可缩放至 200% 且不破坏布局
- [ ] 无每秒闪烁超过 3 次的内容

### 表单
- [ ] 每个输入框具有可见标签
- [ ] 必填字段有标识（不仅依赖颜色）
- [ ] 错误信息具体且与对应字段关联
- [ ] 错误状态不仅通过颜色展示（图标、文本、边框）
- [ ] 表单提交错误被汇总且可聚焦
- [ ] 已知字段使用 autocomplete（例如 `type="email" autocomplete="email"`）

### 内容
- [ ] 声明语言（`<html lang="en">`）
- [ ] 页面具有描述性 `<title>`
- [ ] 链接与周围文本可区分（不仅依赖颜色）
- [ ] 移动端触控目标 ≥ 44x44px
- [ ] 有意义的空状态（非空白屏幕）

## 常见 HTML 模式

### 按钮 vs. 链接

```html
<!-- 操作使用 <button> -->
<button onClick={handleDelete}>Delete Task</button>

<!-- 导航使用 <a> -->
<a href="/tasks/123">View Task</a>

<!-- 切勿使用 div/span 作为按钮 -->
<div onClick={handleDelete}>Delete</div>  <!-- 错误做法 -->
```

### 表单标签

```html
<!-- 显式标签关联 -->
<label htmlFor="email">Email address</label>
<input id="email" type="email" required />

<!-- 隐式包裹 -->
<label>
  Email address
  <input type="email" required />
</label>

<!-- 隐藏标签（优先使用可见标签） -->
<input type="search" aria-label="Search tasks" />
```

### ARIA Roles

```html
<!-- 导航 -->
<nav aria-label="Main navigation">...</nav>
<nav aria-label="Footer links">...</nav>

<!-- 状态消息 -->
<div role="status" aria-live="polite">Task saved</div>

<!-- 警告消息 -->
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

## 测试工具

```bash
# 自动化审计
npx axe-core          # 程序化无障碍测试
npx pa11y             # CLI 无障碍检查器

# 浏览器内
# Chrome DevTools → Lighthouse → Accessibility
# Chrome DevTools → Elements → Accessibility tree

# 屏幕阅读器测试
# macOS: VoiceOver (Cmd + F5)
# Windows: NVDA（免费）或 JAWS
# Linux: Orca
```

## 快速参考：ARIA Live Regions

| 值 | 行为 | 适用场景 |
|-------|----------|---------|
| `aria-live="polite"` | 下次空闲时播报 | 状态更新、保存确认 |
| `aria-live="assertive"` | 立即播报 | 错误、时间敏感警告 |
| `role="status"` | 同 `polite` | 状态消息 |
| `role="alert"` | 同 `assertive` | 错误消息 |

## 常见反模式

| 反模式 | 问题 | 修复方案 |
|---|---|---|
| `div` 作为按钮 | 不可聚焦，无键盘支持 | 使用 `<button>` |
| 缺少 `alt` 文本 | 图片对屏幕阅读器不可见 | 添加描述性 `alt` |
| 仅依赖颜色的状态 | 色盲用户无法识别 | 添加图标、文本或图案 |
| 自动播放媒体 | 令人困惑且无法停止 | 添加控件，不自动播放 |
| 无 ARIA 的自定义下拉框 | 键盘/屏幕阅读器无法使用 | 使用原生 `<select>` 或正确的 ARIA listbox |
| 移除焦点 outline | 用户无法看到当前位置 | 样式化 outline，而非移除 |
| 空链接/按钮 | 播报"链接"但无描述 | 添加文本或 `aria-label` |
| `tabindex > 0` | 破坏自然 Tab 顺序 | 仅使用 `tabindex="0"` 或 `-1` |
