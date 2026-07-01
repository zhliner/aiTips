# 可视化伴侣指南

基于浏览器的可视化头脑风暴伴侣，用于展示模型、图表和选项。

## 何时使用

按问题决策，而不是按会话。检验标准：**用户通过看到它比阅读它理解得更好吗？**

**使用浏览器** 当内容本身是可视化的时候：

- **UI 模型** — 线框图、布局、导航结构、组件设计
- **架构图** — 系统组件、数据流、关系图
- **并排视觉对比** — 比较两个布局、两个配色方案、两个设计方向
- **设计润色** — 当问题涉及外观和感觉、间距、视觉层次时
- **空间关系** — 以图表形式呈现的状态机、流程图、实体关系

**使用终端** 当内容是文本或表格的时候：

- **需求和范围问题** — "X 是什么意思？"、"哪些功能在范围内？"
- **概念性 A/B/C 选择** — 在用文字描述的方法之间选择
- **权衡列表** — 优点/缺点、对比表格
- **技术决策** — API 设计、数据建模、架构方法选择
- **澄清问题** — 任何答案是文字而非视觉偏好的问题

关于 UI 主题的问题不自动是可视化问题。"你想要什么类型的向导？"是概念性的——使用终端。"这些向导布局中哪个感觉合适？"是可视化的——使用浏览器。

## 工作原理

服务器监视一个目录中的 HTML 文件，并将最新的文件提供给浏览器。你将 HTML 内容写入 `screen_dir`，用户在浏览器中看到并可以点击选择选项。选择被记录到 `state_dir/events`，你在下一个回合读取它们。

**内容片段 vs 完整文档：** 如果你的 HTML 文件以 `<!DOCTYPE` 或 `<html` 开头，服务器按原样提供（只注入辅助脚本）。否则，服务器自动将你的内容包装在框架模板中——添加页头、CSS 主题、连接状态和所有交互基础设施。**默认编写内容片段。** 只有当你需要完全控制页面时才编写完整文档。

## 启动会话

```bash
# 在用户批准伴侣后启动。--open 在第一个屏幕上自动打开他们的浏览器；
# --project-dir 持久化模型并启用同端口重启。
scripts/start-server.sh --project-dir /path/to/project --open

# 返回：{"type":"server-started","port":52341,
#           "url":"http://localhost:52341/?key=ab12…",
#           "screen_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/content",
#           "state_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/state"}
```

从响应中保存 `screen_dir` 和 `state_dir`。使用 `--open` 时，当你推送第一个屏幕时浏览器会自动打开——你不需要让用户打开它，但仍然分享 URL 作为备选（无头/远程设置不会自动打开）。

**URL 包含会话密钥（`?key=…`）。** 服务器拒绝任何没有它的请求，
所以始终给用户 `url` 字段中的**完整** URL —
永远不要去掉查询字符串，永远不要给出裸的 `http://host:port`。
密钥控制 HTTP 和 WebSocket 访问，使得杂散的浏览器标签页或网络上的其他机器无法读取屏幕或注入事件。首次加载后，
浏览器通过 cookie 记住密钥，因此重新加载和 `/files/*` 资源无需重复它即可工作。

**查找连接信息：** 服务器将其启动 JSON 写入 `$STATE_DIR/server-info`。如果你在后台启动服务器且未捕获 stdout，读取该文件以获取 URL 和端口。使用 `--project-dir` 时，检查 `<project>/.superpowers/brainstorm/` 以获取会话目录。

**注意：** 将项目根目录作为 `--project-dir` 传递，以便模型在 `.superpowers/brainstorm/` 中持久化，并在服务器重启后保留。没有它，文件将放在 `/tmp` 并被清理。提醒用户将 `.superpowers/` 添加到 `.gitignore`（如果还没有的话）。

**按平台启动服务器：**

**Claude Code：**
```bash
# 默认模式有效 — 脚本自身将服务器置于后台。
scripts/start-server.sh --project-dir /path/to/project --open
```

在 Windows 上，脚本自动检测并切换到前台模式（会阻塞工具调用）。在 Bash 工具调用上使用 `run_in_background: true`，使服务器在对话回合间存活，然后在下一个回合读取 `$STATE_DIR/server-info` 获取 URL 和端口。

**Codex：**
```bash
# Codex 会回收后台进程。脚本自动检测 CODEX_CI 并
# 切换到前台模式。正常运行 — 不需要额外标志。
scripts/start-server.sh --project-dir /path/to/project --open
```

**Copilot CLI：**
```bash
# 使用 --foreground 并通过 bash 工具以 mode: "async" 启动服务器
# 使进程在回合间存活。捕获返回的 shellId 以供
# 后续使用 read_bash / stop_bash 进行交互。
scripts/start-server.sh --project-dir /path/to/project --open --foreground
```

**其他环境：** 服务器必须在对话回合间在后台持续运行。如果你的环境会回收分离的进程，使用 `--foreground` 并用你平台的背景执行机制启动命令。

如果 URL 在浏览器中无法访问（在远程/容器化设置中常见），绑定非回环主机：

```bash
scripts/start-server.sh \
  --project-dir /path/to/project \
  --host 0.0.0.0 \
  --url-host localhost
```

使用 `--url-host` 控制在返回的 URL JSON 中打印什么主机名。

## 循环

1. **检查服务器存活**，然后**写入 HTML** 到 `screen_dir` 中的新文件：
   - **必需：在引用 URL 或推送屏幕之前确认服务器存活。** 检查 `$STATE_DIR/server-info` 存在且 `$STATE_DIR/server-stopped` 不存在。如果它已关闭，使用**相同的 `--project-dir`** 通过 `start-server.sh` 重启它 — 它重用相同端口，所以用户打开的标签页会自动重新连接（服务器关闭时显示"已暂停"覆盖层），你不需要发送新 URL。服务器在 4 小时空闲后自动退出（可通过 `--idle-timeout-minutes` 配置）。
   - 使用语义化文件名：`platform.html`、`visual-style.html`、`layout.html`
   - **永远不要重用文件名** — 每个屏幕使用新文件
   - 使用文件创建工具 — **不要使用 cat/heredoc**（会向终端转储噪音）
   - 服务器自动提供最新文件

2. **告诉用户预期什么并结束回合：**
   - 提醒他们 URL（每一步，不只是第一次）
   - 给出屏幕上内容的简短文字摘要（例如，"展示首页的 3 种布局选项"）
   - 请他们在终端中回复："看看然后告诉我你的想法。如果你想选择某个选项，可以点击选择。"

3. **在你的下一个回合** — 当用户在终端回复后：
   - 如果存在 `$STATE_DIR/events`，读取它 — 这包含用户的浏览器交互（点击、选择）作为 JSON 行
   - 与用户的终端文本合并以获得完整画面
   - 终端消息是主要反馈；`state_dir/events` 提供结构化交互数据

4. **迭代或推进** — 如果反馈改变了当前屏幕，写入新文件（例如，`layout-v2.html`）。只有当前步骤通过验证后才进入下一个问题。

5. **返回终端时卸载** — 当下一步不需要浏览器时（例如，澄清问题、权衡讨论），推送等待屏幕以清除过时内容：

   ```html
   <!-- filename: waiting.html（或 waiting-2.html 等） -->
   <div style="display:flex;align-items:center;justify-content:center;min-height:60vh">
     <p class="subtitle">在终端中继续...</p>
   </div>
   ```

   这防止用户在对话已转移时盯着已解决的选择。当下一个可视化问题出现时，像往常一样推送新内容文件。

6. 重复直到完成。

## 编写内容片段

只编写放入页面内部的内容。服务器自动将其包装在框架模板中（页头、主题 CSS、连接状态和所有交互基础设施）。

**最简示例：**

```html
<h2>哪种布局更好？</h2>
<p class="subtitle">考虑可读性和视觉层次</p>

<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>单栏</h3>
      <p>干净、专注的阅读体验</p>
    </div>
  </div>
  <div class="option" data-choice="b" onclick="toggleSelect(this)">
    <div class="letter">B</div>
    <div class="content">
      <h3>双栏</h3>
      <p>侧边栏导航配合主内容</p>
    </div>
  </div>
</div>
```

就这样。不需要 `<html>`、CSS 或 `<script>` 标签。服务器提供了所有这些。

## 可用的 CSS 类

框架模板为你的内容提供了这些 CSS 类：

### 选项（A/B/C 选择）

```html
<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>标题</h3>
      <p>描述</p>
    </div>
  </div>
</div>
```

**多选：** 向容器添加 `data-multiselect` 使用户可以选择多个选项。每次点击切换项目被选中的样式。

```html
<div class="options" data-multiselect>
  <!-- 相同的选项标记 — 用户可以选择/取消选择多个 -->
</div>
```

### 卡片（可视化设计）

```html
<div class="cards">
  <div class="card" data-choice="design1" onclick="toggleSelect(this)">
    <div class="card-image"><!-- 模型内容 --></div>
    <div class="card-body">
      <h3>名称</h3>
      <p>描述</p>
    </div>
  </div>
</div>
```

### 模型容器

```html
<div class="mockup">
  <div class="mockup-header">预览：仪表盘布局</div>
  <div class="mockup-body"><!-- 你的模型 HTML --></div>
</div>
```

### 分屏视图（并排）

```html
<div class="split">
  <div class="mockup"><!-- 左 --></div>
  <div class="mockup"><!-- 右 --></div>
</div>
```

### 优点/缺点

```html
<div class="pros-cons">
  <div class="pros"><h4>优点</h4><ul><li>好处</li></ul></div>
  <div class="cons"><h4>缺点</h4><ul><li>缺点</li></ul></div>
</div>
```

### 模型元素（线框图构建块）

```html
<div class="mock-nav">Logo | 首页 | 关于 | 联系</div>
<div style="display: flex;">
  <div class="mock-sidebar">导航</div>
  <div class="mock-content">主内容区域</div>
</div>
<button class="mock-button">操作按钮</button>
<input class="mock-input" placeholder="输入字段">
<div class="placeholder">占位符区域</div>
```

### 排版和分区

- `h2` — 页面标题
- `h3` — 章节标题
- `.subtitle` — 标题下方的次要文本
- `.section` — 带底部边距的内容块
- `.label` — 小号大写标签文本

## 浏览器事件格式

当用户在浏览器中点击选项时，他们的交互被记录到 `$STATE_DIR/events`（每行一个 JSON 对象）。当你推送新屏幕时，该文件会自动清除。

```jsonl
{"type":"click","choice":"a","text":"选项 A - 简单布局","timestamp":1706000101}
{"type":"click","choice":"c","text":"选项 C - 复杂网格","timestamp":1706000108}
{"type":"click","choice":"b","text":"选项 B - 混合","timestamp":1706000115}
```

完整的事件流显示了用户的探索路径——他们可能在最终决定前点击多个选项。最后的 `choice` 事件通常是最终选择，但点击模式可以揭示值得询问的犹豫或偏好。

如果 `$STATE_DIR/events` 不存在，用户没有与浏览器交互——仅使用他们的终端文本。

## 设计技巧

- **将保真度与问题匹配** — 布局用线框图，润色问题用精细化设计
- **在每个页面解释问题** — "哪个布局感觉更专业？"而不是仅仅"选一个"
- **在推进之前迭代** — 如果反馈改变了当前屏幕，写入新版本
- **每个屏幕最多 2-4 个选项**
- **当重要时使用真实内容** — 对于摄影作品集，使用真实图片（Unsplash）。占位内容掩盖设计问题。
- **保持模型简单** — 专注于布局和结构，而非像素级完美的设计

## 文件命名

- 使用语义化名称：`platform.html`、`visual-style.html`、`layout.html`
- 永远不要重用文件名——每个屏幕必须是一个新文件
- 对于迭代：追加版本后缀，如 `layout-v2.html`、`layout-v3.html`
- 服务器按修改时间提供最新文件

## 清理

```bash
scripts/stop-server.sh $SESSION_DIR
```

如果会话使用了 `--project-dir`，模型文件在 `.superpowers/brainstorm/` 中持久化以供以后参考。只有 `/tmp` 会话在停止时被删除。

## 参考

- 框架模板（CSS 参考）：`scripts/frame-template.html`
- 辅助脚本（客户端）：`scripts/helper.js`
