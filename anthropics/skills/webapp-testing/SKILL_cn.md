---
name: webapp-testing
description: Toolkit for interacting with and testing local web applications using Playwright. Supports verifying frontend functionality, debugging UI behavior, capturing browser screenshots, and viewing browser logs.
license: Complete terms in LICENSE.txt
---

# Web Application Testing（Web 应用测试）

要测试本地 Web 应用，请编写原生 Python Playwright 脚本。

**可用辅助脚本**：
- `scripts/with_server.py` - 管理服务器生命周期（支持多个服务器）

**始终先运行 `--help` 查看用法**。在尝试运行脚本并确认确实需要自定义方案之前，不要阅读源码。这些脚本可能非常庞大，会污染上下文窗口。它们的设计目的是作为黑盒脚本直接调用，而非被加载到上下文窗口中。

## 决策树：选择你的方案

```
用户任务 → 是否为静态 HTML？
    ├─ 是 → 直接读取 HTML 文件以识别选择器
    │         ├─ 成功 → 使用识别到的选择器编写 Playwright 脚本
    │         └─ 失败/不完整 → 按动态应用处理（见下方）
    │
    └─ 否（动态 Web 应用） → 服务器是否已在运行？
        ├─ 否 → 运行：python scripts/with_server.py --help
        │        然后使用辅助脚本 + 编写简化的 Playwright 脚本
        │
        └─ 是 → 先侦察再行动：
            1. 导航并等待 networkidle
            2. 截图或检查 DOM
            3. 从渲染后的状态中识别选择器
            4. 使用发现的选择器执行操作
```

## 示例：使用 with_server.py

启动服务器前，先运行 `--help` 查看用法，然后使用辅助脚本：

**单个服务器：**
```bash
python scripts/with_server.py --server "npm run dev" --port 5173 -- python your_automation.py
```

**多个服务器（如后端 + 前端）：**
```bash
python scripts/with_server.py \
  --server "cd backend && python server.py" --port 3000 \
  --server "cd frontend && npm run dev" --port 5173 \
  -- python your_automation.py
```

创建自动化脚本时，只需包含 Playwright 逻辑（服务器由辅助脚本自动管理）：
```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True) # 始终以无头模式启动 chromium
    page = browser.new_page()
    page.goto('http://localhost:5173') # 服务器已在运行并就绪
    page.wait_for_load_state('networkidle') # 关键：等待 JavaScript 执行完成
    # ... 你的自动化逻辑
    browser.close()
```

## 先侦察再行动模式

1. **检查渲染后的 DOM**：
   ```python
   page.screenshot(path='/tmp/inspect.png', full_page=True)
   content = page.content()
   page.locator('button').all()
   ```

2. **从检查结果中识别选择器**

3. **使用发现的选择器执行操作**

## 常见陷阱

❌ **不要**在动态应用中等待 `networkidle` 之前就检查 DOM
✅ **要**在检查之前先等待 `page.wait_for_load_state('networkidle')`

## 最佳实践

- **将内置脚本作为黑盒使用** - 完成任务时，考虑 `scripts/` 目录下的脚本是否能提供帮助。这些脚本可靠地处理常见的复杂工作流，且不会污染上下文窗口。使用 `--help` 查看用法，然后直接调用。
- 同步脚本使用 `sync_playwright()`
- 完成后始终关闭浏览器
- 使用描述性选择器：`text=`、`role=`、CSS 选择器或 ID
- 添加适当的等待：`page.wait_for_selector()` 或 `page.wait_for_timeout()`

## 参考文件

- **examples/** - 展示常见模式的示例：
  - `element_discovery.py` - 发现页面上的按钮、链接和输入框
  - `static_html_automation.py` - 使用 file:// URL 处理本地 HTML
  - `console_logging.py` - 在自动化过程中捕获控制台日志
