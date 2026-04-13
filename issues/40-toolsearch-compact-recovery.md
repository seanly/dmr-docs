# P1: ToolSearch 限制优化 — 可配置搜索次数与 Compact 后工具恢复

## 问题

当前 `toolSearch` 存在两个相互加剧的体验问题：

### 1. 搜索次数限制硬编码且过于严格

```go
// plugins/toolsearch/toolsearch.go:69
const maxSearchesPerTape = 3 // Limit to prevent abuse
```

3 次搜索在简单任务中够用，但在复杂多工具场景（例如：先搜索 web 工具获取信息，再搜索 subagent 委派子任务，再搜索 credentials 读取凭据）中很快耗尽。超出后只能列出所有工具名，但并不自动 discover，用户需要再次搜索才能使用。

### 2. Compact 后工具发现状态被清空

```go
// pkg/agent/compact.go:27
a.ClearDiscoveredTools(tapeName)
```

Compact 时不仅重置搜索计数（合理），还清空了所有已 discover 的工具。这意味着 compact 之后：

- 之前用过的扩展工具突然不可用了
- LLM 必须重新 `toolSearch` 来 re-discover
- 这消耗了新的搜索配额
- 如果用户在 compact 前已经用完 3 次搜索，compact 后只有 3 次机会重新发现之前所有用过的工具

**恶性循环**：长对话 → 触发 compact → 工具丢失 → 需要重新 discover → 搜索次数不够 → agent 能力退化

## 设计方案

### 1. 搜索次数可配置

```toml
# ~/.dmr/config.toml
[agent]
max_tool_searches = 5   # 默认 5（从 3 提高），0 = 无限制
```

```go
// plugins/toolsearch/toolsearch.go
func (p *Plugin) Init(_ context.Context, config map[string]any) error {
    if v, ok := config["max_searches"].(int); ok && v > 0 {
        p.maxSearches = v
    }
    return nil
}
```

### 2. Compact 后自动恢复之前 discover 过的工具

核心思路：compact 时记录当前 discovered tools 列表，compact 完成后自动恢复。

```go
// pkg/agent/compact.go

func (a *Agent) compact(ctx context.Context, tapeName, anchorName string) (string, error) {
    // 记录当前 discovered tools（compact 前）
    previouslyDiscovered := a.GetDiscoveredToolNames(tapeName)
    
    // 清空状态
    a.ClearDiscoveredTools(tapeName)
    
    // 执行 compact
    entries, err := a.tape.Compact(ctx, tape.CompactOpts{...})
    if err != nil {
        return "", err
    }
    
    // 恢复之前 discover 过的工具
    for _, toolName := range previouslyDiscovered {
        a.DiscoverTool(tapeName, toolName)
    }
    slog.Info("compact: restored discovered tools", "count", len(previouslyDiscovered), "tape", tapeName)
    
    return summary, nil
}
```

### 3. 在 compact summary 中注入工具列表

让 LLM 知道哪些工具仍然可用：

```go
func (a *Agent) buildCompactSummaryWithTools(summary string, tools []string) string {
    if len(tools) == 0 {
        return summary
    }
    return fmt.Sprintf("%s\n\n[Previously discovered tools still available: %s]",
        summary, strings.Join(tools, ", "))
}
```

### 4. toolSearch 搜索计数只重置不清零

compact 后重置搜索计数（允许重新搜索），但不浪费搜索次数在恢复已知工具上：

```go
// plugins/toolsearch/toolsearch.go
func (p *Plugin) onDiscoveredToolsCleared(_ context.Context, args ...any) (any, error) {
    // 重置搜索计数（允许新搜索）
    p.searchCountMu.Lock()
    delete(p.searchCount, tapeName)
    delete(p.allListed, tapeName)
    p.searchCountMu.Unlock()
    // 注意：不清除已 discover 的工具——由 agent 层处理恢复
    return nil, nil
}
```

## 实现步骤

- [ ] 将 `maxSearchesPerTape` 改为可配置（插件 config 或 agent config）
- [ ] 在 `Agent` 上添加 `GetDiscoveredToolNames(tapeName)` 方法
- [ ] 修改 `compact()` 逻辑：compact 前记录 → 清空 → compact → 恢复
- [ ] 在 compact summary 中注入已恢复的工具列表
- [ ] 添加配置项文档
- [ ] 测试：验证 compact 前后工具可用性不变

## 代价与风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| 恢复的工具在 compact 后的上下文中不再相关 | 中 | 低 | 工具只是 available，不会被强制使用；LLM 自行判断 |
| 无限制搜索导致工具列表过长 | 低 | 中 | 设合理默认值（5），对 LLM 发送的工具列表做最大长度限制 |
| compact summary 注入工具列表增加 token | 低 | 低 | 一行文本，几十个 token |

## 参考

- DMR `plugins/toolsearch/toolsearch.go` — 当前搜索限制逻辑
- DMR `pkg/agent/compact.go` — compact 时清空 discovered tools
- DMR `pkg/agent/loop.go` — agent 循环中的工具收集逻辑
