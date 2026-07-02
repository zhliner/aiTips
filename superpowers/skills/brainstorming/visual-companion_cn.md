# 可视化伴侣指南

基于浏览器的可视化头脑风暴伴侣，用于展示模型、图表和选项。

## 何时使用

逐题决策，而非逐会话决策。测试标准：**用户通过看比通过读能更好地理解这个内容吗？**

**使用浏览器** 处理内容本身是视觉化的场景：

- **UI 模型**——线框图、布局、导航结构、组件设计
- **架构图**——系统组件、数据流、关系图
- **并排视觉对比**——比较两种布局、两种配色方案、两个设计方向
- **设计润色**——当问题涉及外观和感受、间距、视觉层次时
- **空间关系**——状态机、流程图、实体关系图以图表形式呈现

**使用终端** 处理文本或表格内容：

- **需求和范围问题**——"X 是什么意思？""哪些功能在范围内？"
- **概念性 A/B/C 选择**——在用文字描述的方案之间选择
- **权衡列表**——优缺点、对比表格
- **技术决策**——API 设计、数据建模、架构方案选择
- **澄清性问题**——任何答案是用文字而非视觉偏好来表达的问题

*关于* UI 主题的问题并不自动意味着它是视觉化问题。"你想要哪种类型的向导？" 是概念性问题——使用终端。"这些向导布局中哪个感觉合适？" 是视觉化问题——使用浏览器。

## 工作原理

服务器监听一个目录中的 HTML 文件，并将最新的文件提供给浏览器。你向 `screen_dir` 写入 HTML 内容，用户在浏览器中看到它，可以点击选择选项。选择结果记录到 `state_dir/events`，你在下一轮对话中读取。

**内容片段 vs 完整文档：** 如果你的 HTML 文件以 `<!DOCTYPE` 或 `<html` 开头，服务器按原样提供服务（仅注入辅助脚本）。否则，服务器自动将你的内容包装在框架模板中——添加页眉、CSS 主题、连接状态和所有交互基础设施。**默认情况下编写内容片段。** 只有在需要完全控制页面时才编写完整文档。

## 启动会话

```bash
# 在用户批准伴侣后启动。--open 在首次推送页面时自动打开用户的浏览器；
# --project-dir 持久化模型并支持同端口重启。
scripts/start-server.sh --project-dir /path/to/project --open

# 返回：{"type":"server-started","port":52341,
#           "url":"http://localhost:52341/?key=ab12…",
#           "screen_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/content",
#           "state_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/state"}
```

从响应中保存 `screen_dir` 和 `state_dir`。使用 `--open` 时，浏览器会在你推送第一个页面时自动打开——你不需要让用户打开它，但仍然要分享 URL 作为备选（无头/远程设置不会自动打开）。

**URL 包含会话密钥（`?key=…`）。** 服务器会拒绝任何没有该密钥的请求，
因此始终向用户提供 `url` 字段中的**完整** URL——
永远不要去掉查询字符串，也永远不要提供裸的 `http://host:port`。该
密钥控制 HTTP 和 WebSocket 访问，防止浏览器标签页或网络中的其他机器读取页面或注入事件。首次加载后
浏览器通过 cookie 记住密钥，因此重新加载和 `/files/*` 资源无需重复携带密钥即可正常访问。

**查找连接信息：** 服务器将其启动 JSON 写入 `$STATE_DIR/server-info`。如果你在后台启动了服务器且未捕获标准输出，请读取该文件以获取 URL 和端口。使用 `--project-dir` 时，在 `<project>/.superpowers/brainstorm/` 中查找会话目录。

**注意：** 将项目根目录作为 `--project-dir` 传入，以便模型持久化在 `.superpowers/brainstorm/` 中并在服务器重启后仍然保留。如果不传，文件会存入 `/tmp` 并被清理。提醒用户将 `.superpowers/` 添加到 `.gitignore`（如果尚未添加）。

**按平台启动服务器：**

**Claude Code：**
```bash
# 默认模式可用——脚本自行将服务器放入后台。
scripts/start-server.sh --project-dir /path/to/project --open
```

在 Windows 上，脚本会自动检测并切换到前台模式（这会阻塞工具调用）。在 Bash 工具调用上使用 `run_in_background: true`，以便服务器在对话轮次之间保持存活，然后在下一轮读取 `$STATE_DIR/server-info` 获取 URL 和端口。

**Codex：**
```bash
# Codex 会回收后台进程。脚本自动检测 CODEX_CI 并切换到前台模式。正常运行——无需额外标志。
scripts/start-server.sh --project-dir /path/to/project --open
```

**Copilot CLI：**
```bash
# 使用 --foreground 并通过 bash 工具以 mode: "async" 启动服务器，
# 这样进程可以在轮次之间保持存活。捕获返回的 shellId 以便
# 后续通过 read_bash / stop_bash 与之交互。
scripts/start-server.sh --project-dir /path/to/project --open --foreground
```

**其他环境：** 服务器必须在对话轮次之间在后台持续运行。如果你的环境会回收分离的进程，使用 `--foreground` 并通过你的平台的后台执行机制启动命令。

如果 URL 从你的浏览器无法访问（在远程/容器化设置中常见），绑定非回环主机：

```bash
scripts/start-server.sh \
  --project-dir /path/to/project \
  --host 0.0.0.0 \
  --url-host localhost
```

使用 `--url-host` 控制返回的 URL JSON 中打印的主机名。

## 循环流程

1. **检查服务器是否存活**，然后**将 HTML 写入** `screen_dir` 中的新文件：
   - **必须：在引用 URL 或推送页面前确认服务器存活。** 检查 `$STATE_DIR/server-info` 是否存在且 `$STATE_DIR/server-stopped` 不存在。如果服务器已关闭，使用**相同的 `--project-dir`** 通过 `start-server.sh` 重启它——它会重用相同的端口，因此用户的打开标签页会自动重新连接（服务器关闭期间显示"已暂停"覆盖层），你不需要发送新的 URL。服务器在空闲 4 小时后自动退出（可通过 `--idle-timeout-minutes` 配置）。
   - 使用语义化文件名：`platform.html`、`visual-style.html`、`layout.html`
   - **永远不要重用文件名**——每个页面使用新文件
   - 使用你的文件创建工具——**永远不要使用 cat/heredoc**（会将杂乱的输出倾泻到终端）
   - 服务器自动提供最新文件

2. **告诉用户应该看到什么并结束你的轮次：**
   - 提醒他们 URL（每一步都要，不只是第一次）
   - 给出页面上内容的简要文本摘要（例如"展示了首页的 3 种布局选项"）
   - 请他们在终端中回复："请查看并告诉我你的想法。如果愿意，可以点击选择某个选项。"

3. **在你的下一轮**——用户通过终端回复后：
   - 读取 `$STATE_DIR/events`（如果存在）——其中以 JSON 行格式包含用户在浏览器中的交互记录（点击、选择）
   - 与用户的终端文本合并以获得完整图景
   - 终端消息是主要反馈；`state_dir/events` 提供结构化的交互数据

4. **迭代或推进**——如果反馈改变了当前页面，写入新文件（例如 `layout-v2.html`）。只有当前步骤得到验证后才进入下一个问题。

5. **返回终端时卸载**——当下一步不需要浏览器时（例如澄清性问题、权衡讨论），推送一个等待页面以清除过时内容：

   ```html
   <!-- 文件名：waiting.html（或 waiting-2.html 等） -->
   <div style="display:flex;align-items:center;justify-content:center;min-height:60vh">
     <p class="subtitle">正在终端中继续...</p>
   </div>
   ```

   这样可以防止用户在对话已经推进时还盯着一个已解决的选择。当下一个视觉化问题出现时，照常推送新的内容文件。

6. 重复以上步骤直到完成。

## 编写内容片段

只编写放入页面内部的内容。服务器会自动将其包装在框架模板中（页眉、主题 CSS、连接状态和所有交互基础设施）。

**最简示例：**

```html
<h2>哪种布局更好？</h2>
<p class="subtitle">请考虑可读性和视觉层次</p>

<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>单栏</h3>
      <p>干净、聚焦的阅读体验</p>
    </div>
  </div>
  <div class="option" data-choice="b" onclick="toggleSelect(this)">
    <div class="letter">B</div>
    <div class="content">
      <h3>双栏</h3>
      <p>侧边栏导航与主要内容</p>
    </div>
  </div>
</div>
```

就是这样。不需要 `<html>`、不需要 CSS、不需要 `<script>` 标签。服务器提供所有这些。

## 可用的 CSS 类

框架模板为你的内容提供以下 CSS 类：

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

**多选：** 向容器添加 `data-multiselect` 以允许用户选择多个选项。每次点击切换该项的选中样式。

```html
<div class="options" data-multiselect>
  <!-- 相同的选项标记——用户可以选择/取消选择多个 -->
</div>
```

### 卡片（视觉设计）

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
  <div class="mockup"><!-- 左侧 --></div>
  <div class="mockup"><!-- 右侧 --></div>
</div>
```

### 优点/缺点

```html
<div class="pros-cons">
  <div class="pros"><h4>优点</h4><ul><li>优势</li></ul></div>
  <div class="cons"><h4>缺点</h4><ul><li>劣势</li></ul></div>
</div>
```

### 模型元素（线框图构建块）

```html
<div class="mock-nav">Logo | 首页 | 关于 | 联系</div>
<div style="display: flex;">
  <div class="mock-sidebar">导航</div>
  <div class="mock-content">主要内容区域</div>
</div>
<button class="mock-button">操作按钮</button>
<input class="mock-input" placeholder="输入字段">
<div class="placeholder">占位区域</div>
```

### 排版和分节

- `h2`——页面标题
- `h3`——分节标题
- `.subtitle`——标题下方的辅助文字
- `.section`——带底部边距的内容块
- `.label`——小型大写标签文字

## 浏览器事件格式

当用户在浏览器中点击选项时，他们的交互会被记录到 `$STATE_DIR/events`（每行一个 JSON 对象）。当你推送新页面时，该文件会自动清除。

```jsonl
{"type":"click","choice":"a","text":"选项 A - 简单布局","timestamp":1706000101}
{"type":"click","choice":"c","text":"选项 C - 复杂网格","timestamp":1706000108}
{"type":"click","choice":"b","text":"选项 B - 混合布局","timestamp":1706000115}
```

完整的事件流显示用户的探索路径——他们可能在最终确定之前点击多个选项。最后一个 `choice` 事件通常是最终选择，但点击的模式可以揭示犹豫或值得追问的偏好。

如果 `$STATE_DIR/events` 不存在，说明用户没有与浏览器交互——仅使用他们的终端文本。

## 设计技巧

- **根据问题缩放保真度**——布局问题使用线框图，润色问题使用精修稿
- **在每个页面上解释问题**——"哪种布局感觉更专业？" 而不是只说"选一个"
- **迭代后再推进**——如果反馈改变了当前页面，编写新版本
- **每个页面最多 2-4 个选项**
- **在重要时使用真实内容**——对于摄影作品集，使用实际图片（Unsplash）。占位内容会掩盖设计问题。
- **保持模型简洁**——关注布局和结构，而非像素级完美的设计

## 文件命名

- 使用语义化名称：`platform.html`、`visual-style.html`、`layout.html`
- 永远不要重用文件名——每个页面必须是新文件
- 对于迭代版本：追加版本后缀，如 `layout-v2.html`、`layout-v3.html`
- 服务器按修改时间提供最新文件

## 清理

```bash
scripts/stop-server.sh $SESSION_DIR
```

如果会话使用了 `--project-dir`，模型文件保留在 `.superpowers/brainstorm/` 中供后续参考。只有 `/tmp` 会话在停止时会被删除。

## 参考

- 框架模板（CSS 参考）：`scripts/frame-template.html`
- 辅助脚本（客户端）：`scripts/helper.js`
