---
name: webapp-testing
description: 使用 Playwright 与本地 Web 应用程序交互和测试的工具包。支持验证前端功能、调试 UI 行为、捕获浏览器截图和查看浏览器日志。必须通过 CDP 连接到外部已打开的 Chromium 浏览器进行测试。
license: 完整条款见 LICENSE.txt
---

# Web 应用程序测试

要测试本地 Web 应用程序，编写原生 Python Playwright 脚本。必须通过 CDP 连接到 `http://localhost:9222` 上的 Chromium 浏览器，如果连接失败将直接中断执行。

**可用的辅助脚本**：
- `scripts/with_server.py` - 管理服务器生命周期（支持多个服务器）

**始终先使用 `--help` 运行脚本**来查看用法。在尝试运行脚本之前，不要阅读源代码，除非发现确实需要定制化解决方案。这些脚本可能非常大，因此会污染上下文窗口。它们的存在是为了作为黑盒脚本直接调用，而不是被导入到上下文窗口中。

## 决策树：选择您的方法

```
用户任务 → 是否为静态 HTML？
    ├─ 是 → 直接读取 HTML 文件以识别选择器
    │         ├─ 成功 → 使用选择器编写 Playwright 脚本
    │         └─ 失败/不完整 → 视为动态（见下文）
    │
    └─ 否（动态 webapp）→ 确保 Chromium 浏览器在 localhost:9222 上运行 CDP
            ├─ 是 → 侦察-执行模式：
            │        1. 连接到 CDP 端点
            │        2. 导航并等待 networkidle
            │        3. 截图或检查 DOM
            │        4. 从渲染状态识别选择器
            │        5. 使用发现的选择器执行操作
            │
            └─ 否 → 连接失败，直接中断执行并提示用户启动 CDP 浏览器
```

## 示例：使用 with_server.py

要启动服务器，运行 `--help`，然后使用辅助工具：

**单个服务器：**
```bash
python scripts/with_server.py --server "npm run dev" --port 5173 -- python your_automation.py
```

**多个服务器（例如：后端 + 前端）：**
```bash
python scripts/with_server.py \
  --server "cd backend && python server.py" --port 3000 \
  --server "cd frontend && npm run dev" --port 5173 \
  -- python your_automation.py
```

要创建自动化脚本，仅包含 Playwright 逻辑（服务器会自动管理）：

**必须使用 CDP 连接到外部 Chromium 浏览器**：
```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    try:
        # 连接到通过 CDP 在 localhost:9222 上运行的已打开 Chromium
        browser = p.chromium.connect_over_cdp('http://localhost:9222')
        # 使用现有的浏览器上下文或创建新的页面
        page = browser.new_page()
        page.goto('http://localhost:5173') # 服务器已运行并准备就绪
        page.wait_for_load_state('networkidle') # 关键：等待 JS 执行
        # ... 自动化逻辑
        # 注意：不要关闭已连接的浏览器，只需断开连接
        browser.close()
    except Exception as e:
        print(f"❌ 无法连接到 CDP 浏览器: {e}")
        print("请确保 Chromium 浏览器已在 http://localhost:9222 上运行 CDP")
        exit(1)
```

## 侦察-执行模式

1. **检查渲染的 DOM**：
   ```python
   page.screenshot(path='/tmp/inspect.png', full_page=True)
   content = page.content()
   page.locator('button').all()
   ```

2. 从检查结果中**识别选择器**

3. 使用发现的选择器**执行操作**

## 常见陷阱

❌ **不要**在等待 `networkidle` 之前检查动态应用的 DOM
✅ **应该**在检查前等待 `page.wait_for_load_state('networkidle')`

## 最佳实践

- **将捆绑脚本作为黑盒使用** - 要完成任务，考虑 `scripts/` 中可用的脚本是否能帮助。这些脚本可靠地处理常见、复杂的工作流，而不会污染上下文窗口。使用 `--help` 查看用法，然后直接调用。
- 对同步脚本使用 `sync_playwright()`
- 必须包含 CDP 连接失败的错误处理，连接失败时直接中断执行
- 完成时断开浏览器连接（不要关闭外部浏览器）
- 使用描述性选择器：`text=`、`role=`、CSS 选择器或 ID
- 添加适当的等待：`page.wait_for_selector()` 或 `page.wait_for_timeout()`

## CDP 连接要求

**前提条件**：
- Chromium 浏览器已在 `http://localhost:9222` 上运行 CDP
- 连接失败时脚本必须直接中断执行并给出明确提示
- 自动化脚本会影响用户当前的浏览器状态和会话

**优势**：
- 在真实浏览器环境中测试，避免 headless 模式差异
- 可利用已有的浏览器会话、扩展和登录状态
- 便于调试，可同时观察自动化操作和手动操作
- 支持现有的用户 Cookie 和认证状态

## 参考文件

- **references/** - 显示常见模式的示例：
  - `element_discovery.py` - 发现页面上的按钮、链接和输入
  - `static_html_automation.py` - 对本地 HTML 使用 file:// URL
  - `console_logging.py` - 在自动化期间捕获控制台日志