# P1: 浏览器自动化 — MCP 桥接 Playwright

> 灵感来源：Hermes Agent `browser_tool.py`（45KB+ 浏览器自动化工具集）+ MCP 生态系统
>
> DMR 已有 MCP 插件，只需配置即可获得浏览器自动化能力，零代码开发。

## 问题

DMR 当前只有 `webFetch`（HTTP 抓取 + 可选 JavaScript 渲染）和 `webSearch`（DuckDuckGo 搜索），缺乏交互式浏览器操作能力：

| 场景 | 当前行为 | 期望行为 |
|------|---------|---------|
| 登录后才能访问的页面 | 无法处理 | 自动填表登录 |
| SPA 单页应用 | JS 渲染可能不完整 | 完整浏览器环境 |
| 需要点击、滚动、导航的操作 | 不支持 | 完整交互操作 |
| 截图并分析页面效果 | 不支持 | 截图 + 视觉分析（配合 #31） |

## Hermes Agent 的做法

Hermes Agent 内建了一个完整的浏览器自动化工具集（`tools/browser_tool.py`，45KB+）：

```
browser_navigate  — 导航到 URL
browser_click     — 点击元素
browser_type      — 输入文本
browser_scroll    — 滚动页面
browser_snapshot  — 获取页面 DOM 快照
browser_console   — 执行 JavaScript
browser_press     — 按下按键
browser_back      — 后退导航
browser_get_images — 获取页面图片
browser_vision    — 截图 + 视觉分析
```

这是 45KB+ 的内建代码。对 DMR 来说，这种重量级集成**不是正确的路径**。

## DMR 的实现方案

### 核心思路：MCP 桥接，零代码

DMR 已有完整的 MCP 插件（`plugins/mcp/mcp.go`），可以桥接任何 MCP 服务器。浏览器自动化只需**配置**即可获得。

### 方案 1：Playwright MCP Server（推荐）

```toml
# ~/.dmr/config.toml

[[plugins]]
name = "mcp"
enabled = true

[plugins.config.servers.playwright]
command = "npx"
args = ["@anthropic-ai/mcp-playwright"]
```

安装依赖：

```bash
npm install -g @anthropic-ai/mcp-playwright
npx playwright install chromium
```

配置完成后，DMR 自动发现以下工具（通过 `plugins/mcp/bridge.go:39-75` 的工具桥接机制）：

```
mcp_playwright_navigate       — 导航到 URL
mcp_playwright_click          — 点击元素
mcp_playwright_fill           — 填写表单
mcp_playwright_screenshot     — 截图
mcp_playwright_evaluate       — 执行 JavaScript
mcp_playwright_hover          — 悬停
mcp_playwright_select_option  — 下拉选择
mcp_playwright_get_text       — 获取文本内容
```

### 方案 2：Puppeteer MCP Server（备选）

```toml
[plugins.config.servers.puppeteer]
command = "npx"
args = ["-y", "@anthropic-ai/mcp-puppeteer"]
```

### MCP 桥接原理

DMR 的 MCP 桥接在 `plugins/mcp/bridge.go` 中自动完成：

```go
// plugins/mcp/bridge.go:49-75 — 已有代码

func bridgeTool(serverName string, mcpTool mcp.Tool, conn *serverConn) *tool.Tool {
    return &tool.Tool{
        Name:        fmt.Sprintf("mcp_%s_%s", serverName, mcpTool.Name),
        Description: mcpTool.Description,
        Parameters:  inputSchemaToMap(mcpTool.InputSchema),
        Handler: func(ctx *tool.ToolContext, args map[string]any) (any, error) {
            result, err := conn.client.CallTool(ctx.Context, mcpTool.Name, args)
            if err != nil {
                return nil, err
            }
            if result.IsError {
                return map[string]any{"error": extractText(result)}, nil
            }
            return extractText(result), nil
        },
        Group: tool.ToolGroupMCP,  // 默认延迟加载
    }
}
```

### 工具发现流程

```
1. DMR 启动
2. MCP 插件 Init() → 连接 Playwright MCP server
3. MCP ListTools RPC → 发现所有可用工具
4. bridgeTools() → 转换为 DMR tool.Tool
5. 注册为 Extended 工具（延迟发现，按需加载）
6. Agent 执行 toolSearch("browser") → 发现 mcp_playwright_* 工具
7. 正常使用
```

### 与视觉分析集成（配合 #31）

```
用户: "检查一下 localhost:3000 的页面布局是否正确"

Agent 执行流程:
1. mcp_playwright_navigate → http://localhost:3000
2. mcp_playwright_screenshot → /tmp/screenshot.png
3. visionAnalyze(image="/tmp/screenshot.png", prompt="检查页面布局...")
4. 返回分析结果
```

### OPA 策略配置

浏览器操作属于高风险（可能访问恶意网站、泄露凭据），建议配置审批策略：

```rego
# ~/.dmr/policies/browser.rego

package dmr.policy

import rego.v1

# 允许本地 URL 直接访问
default allow_browser = false

allow_browser if {
    input.tool == "mcp_playwright_navigate"
    url := input.args.url
    startswith(url, "http://localhost")
}

allow_browser if {
    input.tool == "mcp_playwright_navigate"
    url := input.args.url
    startswith(url, "http://127.0.0.1")
}

# 外部 URL 需要审批
require_approval if {
    input.tool == "mcp_playwright_navigate"
    url := input.args.url
    not startswith(url, "http://localhost")
    not startswith(url, "http://127.0.0.1")
}

# 截图始终允许
allow_browser if {
    input.tool == "mcp_playwright_screenshot"
}
```

### 允许规则简化

```toml
# config.toml — OPA 插件配置
[[plugins]]
name = "opa_policy"
enabled = true

[plugins.config]
allow_rules = [
    "mcp_playwright_screenshot:*",
    "mcp_playwright_get_text:*",
    "mcp_playwright_evaluate:*",
]
```

## 解决的问题

- **交互式 Web 操作**：登录、表单填写、SPA 导航
- **页面截图**：结合 #31 视觉分析实现完整的 Web 测试闭环
- **零代码开发**：利用现有 MCP 插件，只需配置
- **安全控制**：通过 OPA 策略限制可访问的 URL 范围
- **生态复用**：受益于 MCP 生态的持续更新（Playwright、Puppeteer 等）

## 代价与风险

| 代价 | 评估 |
|------|------|
| **外部依赖** | 需要安装 Node.js + Playwright（`npm install`） |
| **资源消耗** | Chromium 进程占用 ~200-500MB 内存 |
| **启动延迟** | MCP server 首次启动需 5-10 秒 |
| **安全风险** | 浏览器可访问任意 URL → OPA 策略 + 审批缓解 |
| **维护成本** | 零（依赖 MCP 服务器维护者） |

## 可选增强

### 1. webFetch 集成增强

当前 `webFetch` 的 JavaScript 渲染已使用 Chromium（`plugins/webtool/plugin.go:27-29`）。可以考虑让 `webFetch` 的 renderJavaScript 模式在有 Playwright MCP 可用时自动使用它，避免重复的 Chromium 实例。

### 2. 自动截图 + 视觉反馈

在 `AfterAgentRun` 中，如果检测到浏览器操作，自动截图并注入下一轮上下文：

```go
// 伪代码
func afterBrowserAction(tapeName string) {
    // 如果本轮使用了 mcp_playwright_* 工具
    // 自动调用 mcp_playwright_screenshot
    // 调用 visionAnalyze 获取页面状态描述
    // 注入 tape 作为上下文
}
```

## 参考

- Hermes Agent browser_tool：`tools/browser_tool.py`（45KB+）
- DMR MCP 插件：`plugins/mcp/mcp.go:12-62`
- DMR MCP 桥接：`plugins/mcp/bridge.go:39-75`
- DMR MCP 客户端：`plugins/mcp/client.go:32-94`
- DMR OPA 策略引擎：`plugins/opapolicy/opapolicy.go`
- Playwright MCP：`@anthropic-ai/mcp-playwright`（npm）
- DMR webFetch 渲染：`plugins/webtool/plugin.go:27-29`
