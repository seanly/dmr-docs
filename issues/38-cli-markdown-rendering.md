# P2: CLI Markdown 渲染 — 终端富文本输出

> **Status**: ✅ **已完成** (commit: `68510fe`)

## 问题

当前 `plugins/cli/renderer.go` 对 LLM 输出采用纯文本 + 颜色打印，没有 Markdown 解析和渲染。LLM 返回的 markdown（代码块、表格、列表、标题）原样输出到终端。

典型痛点：

1. **代码块没有语法高亮**：LLM 返回的 ` ```go ` 代码块直接打印原文，包含 ``` 标记，阅读体验差
2. **表格无法对齐**：Markdown 表格在等宽字体终端下勉强可读，但没有边框和高亮
3. **链接不可点击**：URL 混在文本中，无法识别
4. **嵌套列表缩进丢失**：纯 `fmt.Println` 不理解 Markdown 缩进语义

### 现状

```go
// plugins/cli/renderer.go:45
func (r *Renderer) PrintAssistant(text string) {
    r.assistant.Print("Assistant> ")
    fmt.Println(text)  // 原样打印，无渲染
    fmt.Println()
}
```

## 设计方案

### 引入 glamour 终端 Markdown 渲染

使用 `charmbracelet/glamour` 库（Go 生态最成熟的终端 Markdown 渲染器），在输出前将 LLM 文本渲染为带颜色和格式的终端文本。

```go
// plugins/cli/renderer.go

import "github.com/charmbracelet/glamour"

func (r *Renderer) renderMarkdown(text string) string {
    renderer, err := glamour.NewTermRenderer(
        glamour.WithAutoStyle(),       // 自动适配深色/浅色终端
        glamour.WithWordWrap(termWidth()),
    )
    if err != nil {
        return text  // fallback 到原文
    }
    out, err := renderer.Render(text)
    if err != nil {
        return text
    }
    return out
}

func (r *Renderer) PrintAssistant(text string) {
    r.assistant.Print("Assistant> ")
    fmt.Print(r.renderMarkdown(text))
    fmt.Println()
}
```

### 配置化

```toml
# ~/.dmr/config.toml
[[plugins]]
name = "cli"
enabled = true
[plugins.config]
markdown_render = true    # 默认开启，可关闭
word_wrap = 0             # 0 = 自动检测终端宽度
tool_result_max = 200     # 工具结果显示最大字符数
```

对于 pipe 场景（`dmr run "..." | grep`），应自动检测是否为 TTY，非 TTY 时禁用渲染。

### 工具调用结果的渲染

工具结果（`PrintToolCall`）当前只显示 200 字符截断。改进：

```go
func (r *Renderer) PrintToolResult(name, result string) {
    if len(result) > r.maxToolResultLen {
        // 显示截断摘要 + 提示完整内容在 tape 中
        fmt.Printf("    => [%d chars, showing first %d]\n", len(result), r.maxToolResultLen)
        fmt.Println(r.renderMarkdown(result[:r.maxToolResultLen]))
        fmt.Println("    ... (full result in tape)")
    } else {
        fmt.Println(r.renderMarkdown(result))
    }
}
```

## 实现步骤

- [x] 添加 `charmbracelet/glamour` 依赖
- [x] 在 `Renderer` 中初始化 glamour renderer（懒加载，支持 TTY 检测）
- [x] 改造 `PrintAssistant()` 使用 Markdown 渲染
- [x] 改造 `PrintToolCall()` 支持可配置的截断长度
- [x] 添加 `[cli]` 插件配置项
- [x] 非 TTY 场景自动降级为纯文本

## 实现总结

### 提交信息
```
commit 68510fea9457ffbd80216cca12a68bcf385a45c2
feat(cli): add terminal Markdown rendering with glamour
```

### 变更文件
| 文件 | 变更 |
|------|------|
| `plugins/cli/renderer.go` | 新增 `RenderConfig`, Markdown 渲染, TTY 检测 |
| `plugins/cli/cli.go` | 在 `Init()` 中解析插件配置 |
| `plugins/cli/chat.go` | 传递配置给 Renderer |
| `config.example.toml` | 添加 cli 插件配置示例 |
| `AGENTS.md` | 添加 CLI Markdown 渲染文档 |
| `CHANGELOG.md` | 添加未发布版本条目 |
| `go.mod/go.sum` | 添加 glamour 依赖 |

### 配置示例
```toml
[[plugins]]
name = "cli"
enabled = true
[plugins.config]
markdown_render = true    # 启用终端 Markdown 渲染（默认：true）
word_wrap = 0             # 换行宽度，0 = 自动检测
tool_result_max = 200     # 工具结果显示最大字符数
```

### 关键特性
1. **自动 TTY 检测**: 管道输出时自动降级为纯文本
2. **懒加载**: glamour renderer 首次使用时初始化
3. **审批无影响**: 审批 UI (`approver.go`) 保持独立
4. **向后兼容**: 无配置时默认启用渲染

### 体积影响
- 二进制大小: 54MB → 61MB (+7MB)

## 代价与风险

| 风险 | 概率 | 影响 | 缓解措施 | 实际结果 |
|------|------|------|----------|----------|
| glamour 增加二进制体积 | 确定 | 低 | dmr 已 54 MB | ✅ 实际 +7MB，可接受 |
| 部分终端不支持 ANSI 渲染 | 低 | 中 | TTY 检测自动降级 | ✅ 已实现 |
| 渲染与流式输出的兼容 | 中 | 中 | 流式模式下按段落/代码块边界缓冲 | ⚠️ 当前非流式，未来可扩展 |
| LLM 输出非标准 Markdown | 低 | 低 | glamour 优雅降级 | ✅ fallback 到原文 |

## 参考

- `charmbracelet/glamour` — 终端 Markdown 渲染库
- `charmbracelet/glow` — 基于 glamour 的 CLI Markdown 阅读器
- DMR `plugins/cli/renderer.go` — 当前渲染实现
- Claude Code CLI — 终端 Markdown 渲染的参考体验
