---
name: browser
description: 控制浏览器执行自动化操作。当用户需要打开网页抓取内容、提取文字或图片、截取网页截图、自动化操作网页（填表单、点按钮、翻页）、下载网页资源时使用此技能。即使用户只是给了一个 URL 说"帮我看看"或"打开这个"，也应触发此技能。涉及任何需要浏览器交互才能完成的任务（如页面需要登录、动态加载、JavaScript 渲染）都应使用。
user-invokable: true
---

# 浏览器自动化控制

通过多层浏览器工具完成网页操作。优先连接用户已登录的 Chrome 复用登录态，如果不可用则自动降级到 CLI 或 HTTP 工具。

## 工具选择（分层 fallback）

按优先级依次尝试，用第一个可用的层级完成任务。不要因为高优先级工具不可用就放弃——总有一层能用。

### Tier 1：Playwright MCP（最佳，完整浏览器控制）

检查是否有 `browser_navigate`、`browser_snapshot` 等 MCP 工具。如果有，直接使用。

如果没有但任务需要完整浏览器控制（登录态、复杂交互），告知用户可一次性配置：
```bash
claude mcp add playwright -- npx @playwright/mcp@latest --browser chrome --cdp-url http://localhost:9222
```
同时需要 Chrome 调试端口开启（任选其一）：
- 安装 Playwright MCP Bridge 扩展（全自动）
- Chrome 地址栏 `chrome://inspect/#remote-debugging`（M144+）
- 命令行：`chrome.exe --remote-debugging-port=9222 --remote-allow-origins=* --user-data-dir=<临时目录>`
  - 注意：如果默认 profile 下调试端口不开（企业策略限制），需要用 `--user-data-dir` 指定临时目录

然后继续用 Tier 2 或 Tier 3 完成当前任务，不要停下来等用户配置。

### Tier 2：Playwright CLI / 脚本（截图、自动化脚本）

通过命令行或脚本操作浏览器。不需要 MCP 配置。有两种方式，哪个能用就用哪个：

**方式 A：npx CLI（快捷，适合截图/PDF）**
```bash
npx playwright screenshot --channel msedge --full-page --wait-for-timeout 3000 "https://example.com" output.png
npx playwright pdf --channel msedge "https://example.com" output.pdf
```
- `--channel msedge` 使用系统已装的 Edge，无需额外下载浏览器
- 如果 npx 不可用，先 `npm install -g playwright` 或 `pip install playwright`

**方式 B：Python 脚本（更灵活，适合复杂操作）**
```python
from playwright.sync_api import sync_playwright
with sync_playwright() as p:
    browser = p.chromium.launch(channel="msedge", headless=True)
    page = browser.new_page()
    page.goto("https://example.com", wait_until="networkidle")
    page.screenshot(path="output.png", full_page=True)
    # 可以执行任意操作：点击、填写、提取内容、等待元素...
    browser.close()
```
- 需要 `pip install playwright && playwright install`（首次使用）
- 比 npx CLI 灵活得多：可以等待特定元素、执行 JS、处理动态内容
- 适合截图、内容提取、自动化表单等 CLI 做不到的操作

### Tier 3：WebFetch / curl + 创造性绕行

对于不需要登录的页面，先尝试直接获取：
- `WebFetch` 一步获取页面内容（自动 HTML→Markdown + AI 提取）
- `curl` 下载文件/图片

**如果遇到 403/JS 渲染等阻塞，不要直接放弃**，尝试以下绕行策略：
- **公开 API**：很多网站有 REST API（如 npm registry `registry.npmjs.org`、GitHub API `api.github.com`），数据比页面更结构化
- **CDN 直链**：图片/资源通常托管在 CDN 上（如 `images.unsplash.com`），可以绕过主站的反爬
- **WebSearch 辅助**：用 WebSearch 搜索目标内容的直接 URL、API 文档、或替代数据源
- **原始内容源**：GitHub 文件用 `raw.githubusercontent.com`，npm 包信息用 registry API

如果所有绕行都失败，告知用户此任务需要 Tier 1/2 并给出配置指引。

## Playwright MCP 交互模型

当 Tier 1 可用时，以**无障碍快照（accessibility snapshot）**为核心交互方式。快照返回带 ref 编号的页面元素树，比截图更快、更准确、更省 token。

基本循环：
1. `browser_snapshot` → 拿到页面结构和元素 ref
2. 用 ref 执行操作（`browser_click ref="e15"`）
3. 再次 `browser_snapshot` 确认结果
4. 重复直到完成

只在需要视觉确认时才用 `browser_take_screenshot`。

## 工作流

每个工作流都标注了所需的最低 Tier。如果当前可用 Tier 不够，向上降级到能用的 Tier 并尽力完成。

### 内容提取（Tier 3 即可，Tier 1 更好）

**Tier 3（公开页面）**：
1. `WebFetch` 获取页面内容，用 prompt 指定要提取的信息

**Tier 1（需要登录或 JS 渲染）**：
1. `browser_navigate` 到目标页面
2. `browser_snapshot` 获取文本内容
3. 需要结构化数据时用 `browser_evaluate` 执行 JS：
   ```javascript
   document.querySelector('article')?.innerText
   [...document.querySelectorAll('img')].map(img => ({ src: img.src, alt: img.alt }))
   [...document.querySelectorAll('table tr')].map(tr => [...tr.cells].map(td => td.textContent))
   ```

### 截图（Tier 2 即可）

依次尝试直到成功：
1. **Tier 1**：`browser_navigate` → `browser_take_screenshot`（支持指定元素截图）
2. **Tier 2A**：`npx playwright screenshot --channel msedge --full-page "<url>" output.png`
3. **Tier 2B**：Python Playwright 脚本（可以等待特定元素加载后再截图，更精确）

截图到特定区域（如 README）时，Tier 2B 的 `page.locator("article").screenshot()` 比全页截图更有用。

### 图片批量下载（多层策略）

**⚠ 批量操作前先报告数量，等用户确认后再执行。**

1. **Tier 1**：`browser_navigate` → `browser_evaluate` 提取图片 URL → `curl` 下载
2. **Tier 2B**：Python Playwright 脚本打开页面、提取 URL、下载
3. **Tier 3 创造性绕行**（主站返回 403 时）：
   - `WebSearch` 搜索该网站的图片 CDN 域名或 API
   - 通过 API/CDN 直接获取图片 URL
   - `curl` 或 `WebFetch` 下载

很多图片网站（Unsplash、Pexels 等）都有公开 CDN，即使主站反爬也能通过 CDN 直链下载。

### 表单填写 / 页面操作（需要 Tier 1）

**⚠ 涉及提交、删除、发送等写操作时，先告知用户具体动作，等确认后再执行。**

1. `browser_snapshot` 识别表单元素
2. 用 `browser_click`、`browser_type`、`browser_select_option` 依次操作
3. 每步后 `browser_snapshot` 确认状态变化

### 翻页采集（需要 Tier 1）

**⚠ 开始前先报告预估页数，等用户确认。**

1. 处理当前页内容
2. snapshot 找"下一页"按钮 ref
3. `browser_click` 翻页，等待加载，重复

## 安全原则

连接的是用户**真实浏览器**，操作不可撤销。上面各工作流中已标注 ⚠ 的地方必须遵守，总结如下：

- **写操作**（提交、删除、发送）→ 告知 + 等确认
- **批量操作**（下载、翻页）→ 报告规模 + 等确认
- **敏感页面**（支付、授权）→ 截图展示，不自行操作
- 不主动关闭标签页、点击弹窗或修改用户未要求改动的内容
