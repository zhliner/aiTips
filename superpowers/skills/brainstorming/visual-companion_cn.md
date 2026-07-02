# Visual Companion 指南

基于浏览器的可视化 brainstorming companion，用于展示 mockup、图表和选项。

## 何时使用

逐问题决定，而非逐 session。判断标准：**用户通过看到它比阅读它理解得更好吗？**

**使用浏览器**，当内容本身是视觉的：

- **UI mockup** — 线框图、布局、导航结构、组件设计
- **架构图** — 系统组件、数据流、关系映射
- **并排视觉对比** — 比较两种布局、两种配色方案、两种设计方向
- **设计打磨** — 当问题关于外观和感觉、间距、视觉层次
- **空间关系** — 状态机、流程图、渲染为图表的实体关系

**使用终端**，当内容是文本或表格的：

- **需求和范围问题** — "X 是什么意思？"、"哪些功能在范围内？"
- **概念性 A/B/C 选择** — 在用文字描述的方案之间选择
- **权衡列表** — 优缺点、对比表
- **技术决策** — API 设计、数据建模、架构方案选择
- **澄清问题** — 任何答案是文字而非视觉偏好的问题

关于 UI 话题的问题不自动成为可视化问题。"你想要什么样的向导？"是概念性的——使用终端。"这些向导布局中哪个感觉对？"是视觉的——使用浏览器。

## 工作原理

服务器监视一个目录中的 HTML 文件，并将最新的文件提供给浏览器。你将 HTML 内容写入 `screen_dir`，用户在浏览器中看到它并可以点击选择选项。选择操作被记录到 `state_dir/events`，你在下一轮读取。

**内容片段 vs 完整文档：** 如果你的 HTML 文件以 `<!DOCTYPE` 或 `<html` 开头，服务器按原样提供（仅注入 helper 脚本）。否则，服务器自动将你的内容包装在 frame 模板中——添加头部、CSS 主题、连接状态和所有交互基础设施。**默认编写内容片段。** 仅在需要完全控制页面时才编写完整文档。

## 启动 Session

```bash
# 在用户批准 companion 后启动。--open 自动在第一个画面打开用户浏览器；
# --project-dir 持久化 mockup 并支持同端口重启。
scripts/start-server.sh --project-dir /path/to/project --open

# 返回：{"type":"server-started","port":52341,
#         "url":"http://localhost:52341/?key=ab12…",
#         "screen_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/content",
#         "state_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/state"}
```

从响应中保存 `screen_dir` 和 `state_dir`。使用 `--open` 时，浏览器在你推送第一个画面时自动打开——你不需要请用户打开它，但仍分享 URL 作为备选（headless/远程设置不会自动打开）。

**URL 包含 session key（`?key=…`）。** 服务器拒绝任何没有它的请求，所以始终给用户提供 `url` 字段中的**完整** URL——永远不要去掉查询字符串，也永远不要给出裸的 `http://host:port`。此 key 控制 HTTP 和 WebSocket 访问，防止 stray 浏览器标签或网络上的其他机器读取画面或注入事件。首次加载后浏览器通过 cookie 记住 key，因此重新加载和 `/files/*` 资源无需重复传递。

**查找连接信息：** 服务器将启动 JSON 写入 `$STATE_DIR/server-info`。如果你在后台启动了服务器且未捕获 stdout，读取该文件获取 URL 和端口。使用 `--project-dir` 时，检查 `<project>/.superpowers/brainstorm/` 找到 session 目录。

**注意：** 传递项目根目录作为 `--project-dir`，这样 mockup 持久化在 `.superpowers/brainstorm/` 中并在服务器重启后保留。不传递的话，文件进入 `/tmp` 并被清理。提醒用户如果 `.superpowers/` 还不在 `.gitignore` 中，应将其添加。

**各平台启动服务器方式：**

**Claude Code：**
```bash
# 默认模式即可——脚本本身将服务器放入后台。
scripts/start-server.sh --project-dir /path/to/project --open
```

在 Windows 上，脚本自动检测并切换到前台模式（会阻塞工具调用）。在 Bash 工具调用上使用 `run_in_background: true`，使服务器在对话轮次间存活，然后在下一轮读取 `$STATE_DIR/server-info` 获取 URL 和端口。

**Codex：**
```bash
# Codex 会终止后台进程。脚本自动检测 CODEX_CI 并切换到前台模式。
# 正常运行即可——无需额外标志。
scripts/start-server.sh --project-dir /path/to/project --open
```

**Copilot CLI：**
```bash
# 使用 --foreground 并通过 bash 工具以 mode: "async" 启动服务器，
# 使进程在轮次间存活。捕获返回的 shellId 以便后续使用
# read_bash / stop_bash 进行交互。
scripts/start-server.sh --project-dir /path/to/project --open --foreground
```

**其他环境：** 服务器必须在后台持续运行，跨对话轮次存活。如果你的环境会终止分离的进程，使用 `--foreground` 并通过你平台的后台执行机制启动命令。

如果 URL 从你的浏览器无法访问（在远程/容器化环境中常见），绑定非回环主机：

```bash
scripts/start-server.sh \
  --project-dir /path/to/project \
  --host 0.0.0.0 \
  --url-host localhost
```

使用 `--url-host` 控制返回的 URL JSON 中打印的主机名。

## 循环

1. **检查服务器存活**，然后**将 HTML 写入** `screen_dir` 中的新文件：
   - **必须：在引用 URL 或推送画面之前确认服务器存活。** 检查 `$STATE_DIR/server-info` 存在且 `$STATE_DIR/server-stopped` 不存在。如果已关闭，使用**相同的 `--project-dir`** 通过 `start-server.sh` 重启——它会复用相同端口，因此用户已打开的标签会自动重连（服务器关闭时显示"paused"覆盖层），你不需要发送新 URL。服务器在空闲 4 小时后自动退出（可通过 `--idle-timeout-minutes` 配置）。
   - 使用语义化文件名：`platform.html`、`visual-style.html`、`layout.html`
   - **永远不要复用文件名** — 每个画面使用新文件
   - 使用你的文件创建工具 — **永远不要使用 cat/heredoc**（会在终端输出噪音）
   - 服务器自动提供最新文件

2. **告诉用户期待什么并结束你的轮次：**
   - 提醒他们 URL（每步都要，不只是第一次）
   - 简要文字总结画面内容（例如"展示首页的 3 种布局选项"）
   - 请他们在终端回复："看一下，告诉我你的想法。如果愿意，可以点击选择选项。"

3. **在你的下一轮** — 用户在终端回复后：
   - 读取 `$STATE_DIR/events`（如果存在）— 包含用户的浏览器交互（点击、选择），格式为 JSON lines
   - 结合用户的终端文本获得完整画面
   - 终端消息是主要反馈；`state_dir/events` 提供结构化交互数据

4. **迭代或推进** — 如果反馈改变当前画面，写入新文件（例如 `layout-v2.html`）。仅在当前步骤验证后才进入下一个问题。

5. **返回终端时卸载** — 当下一步不需要浏览器时（例如澄清问题、权衡讨论），推送等待画面以清除过时内容：

   ```html
   <!-- filename: waiting.html (or waiting-2.html, etc.) -->
   <div style="display:flex;align-items:center;justify-content:center;min-height:60vh">
     <p class="subtitle">Continuing in terminal...</p>
   </div>
   ```

   这防止用户盯着已解决的选择而对话已经继续。当下一个可视化问题出现时，照常推送新内容文件。

6. 重复直到完成。

## 编写内容片段

只写页面内部的内容。服务器自动将其包装在 frame 模板中（头部、主题 CSS、连接状态和所有交互基础设施）。

**最小示例：**

```html
<h2>Which layout works better?</h2>
<p class="subtitle">Consider readability and visual hierarchy</p>

<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>Single Column</h3>
      <p>Clean, focused reading experience</p>
    </div>
  </div>
  <div class="option" data-choice="b" onclick="toggleSelect(this)">
    <div class="letter">B</div>
    <div class="content">
      <h3>Two Column</h3>
      <p>Sidebar navigation with main content</p>
    </div>
  </div>
</div>
```

就是这样。不需要 `<html>`、CSS 或 `<script>` 标签。服务器提供所有这些。

## 可用 CSS 类

frame 模板为你的内容提供以下 CSS 类：

### 选项（A/B/C 选择）

```html
<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>Title</h3>
      <p>Description</p>
    </div>
  </div>
</div>
```

**多选：** 在容器上添加 `data-multiselect` 允许用户选择多个选项。每次点击切换该项的选中样式。

```html
<div class="options" data-multiselect>
  <!-- 相同的选项标记——用户可以选择/取消选择多个 -->
</div>
```

### 卡片（视觉设计）

```html
<div class="cards">
  <div class="card" data-choice="design1" onclick="toggleSelect(this)">
    <div class="card-image"><!-- mockup 内容 --></div>
    <div class="card-body">
      <h3>Name</h3>
      <p>Description</p>
    </div>
  </div>
</div>
```

### Mockup 容器

```html
<div class="mockup">
  <div class="mockup-header">Preview: Dashboard Layout</div>
  <div class="mockup-body"><!-- 你的 mockup HTML --></div>
</div>
```

### 分栏视图（并排）

```html
<div class="split">
  <div class="mockup"><!-- 左侧 --></div>
  <div class="mockup"><!-- 右侧 --></div>
</div>
```

### 优缺点

```html
<div class="pros-cons">
  <div class="pros"><h4>Pros</h4><ul><li>Benefit</li></ul></div>
  <div class="cons"><h4>Cons</h4><ul><li>Drawback</li></ul></div>
</div>
```

### Mock 元素（线框图构建块）

```html
<div class="mock-nav">Logo | Home | About | Contact</div>
<div style="display: flex;">
  <div class="mock-sidebar">Navigation</div>
  <div class="mock-content">Main content area</div>
</div>
<button class="mock-button">Action Button</button>
<input class="mock-input" placeholder="Input field">
<div class="placeholder">Placeholder area</div>
```

### 排版和分区

- `h2` — 页面标题
- `h3` — 分区标题
- `.subtitle` — 标题下方的副标题文本
- `.section` — 带底部间距的内容块
- `.label` — 小号大写标签文本

## 浏览器事件格式

当用户在浏览器中点击选项时，他们的交互被记录到 `$STATE_DIR/events`（每行一个 JSON 对象）。推送新画面时该文件自动清空。

```jsonl
{"type":"click","choice":"a","text":"Option A - Simple Layout","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Complex Grid","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid","timestamp":1706000115}
```

完整事件流显示用户的探索路径——他们可能在最终确定前点击多个选项。最后一个 `choice` 事件通常是最终选择，但点击模式可以揭示值得询问的犹豫或偏好。

如果 `$STATE_DIR/events` 不存在，用户没有与浏览器交互——仅使用他们的终端文本。

## 设计建议

- **根据问题调整保真度** — 布局问题用线框图，打磨问题用精细设计
- **在每个页面解释问题** — "哪个布局感觉更专业？"而非仅仅"选一个"
- **推进前先迭代** — 如果反馈改变当前画面，编写新版本
- **每个画面最多 2-4 个选项**
- **在重要时使用真实内容** — 对于摄影作品集，使用真实图片（Unsplash）。占位符内容会掩盖设计问题。
- **保持 mockup 简洁** — 关注布局和结构，而非像素级完美设计

## 文件命名

- 使用语义化名称：`platform.html`、`visual-style.html`、`layout.html`
- 永远不要复用文件名——每个画面必须是新文件
- 迭代时：附加版本后缀如 `layout-v2.html`、`layout-v3.html`
- 服务器按修改时间提供最新文件

## 清理

```bash
scripts/stop-server.sh $SESSION_DIR
```

如果 session 使用了 `--project-dir`，mockup 文件持久化在 `.superpowers/brainstorm/` 中供后续参考。只有 `/tmp` session 会在停止时被删除。

## 参考

- Frame 模板（CSS 参考）：`scripts/frame-template.html`
- Helper 脚本（客户端）：`scripts/helper.js`
