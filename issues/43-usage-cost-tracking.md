# P1: 执行成本与进度追踪 — Token 消耗与 Agent 步骤可视化

## 问题

DMR 重度用户每天发送大量 LLM 请求，但完全无法感知：

1. **当前会话消耗了多少 token / 费用**：不同模型定价不同，用户无法做成本预算
2. **Agent 循环进度**：当前跑到第几步、最大步数限制是多少、离 compact 还有多远
3. **Compact 状态**：是否即将触发、上次 compact 是什么时候
4. **累计统计**：本月总消耗、每个 tape 的消耗分布

### 现状

- `pkg/agent/loop.go` 在每次 LLM 调用后已经提取 `usage` 信息（`extractUsage()`）
- token 数据被记录到 slog 日志中，但用户在 CLI 看不到
- 没有任何持久化的 token 统计

## 设计方案

### 1. 实时进度显示

在 CLI 的 prompt 或工具调用间隙显示进度信息：

```text
You> 帮我分析这个代码库
  [step 1/20 | 1.2k tokens | ~$0.003]
  tool> shell git log --oneline -10
    => ...
  [step 2/20 | 3.8k tokens | ~$0.009]
  tool> fsRead src/main.go
    => ...
  [step 3/20 | 8.1k tokens | ~$0.021 | compact at ~32k]
Assistant> ...
  [total: 12.4k tokens | $0.031 | 4 steps]
```

### 2. Usage Tracker

```go
// pkg/agent/usage_tracker.go

type UsageTracker struct {
    mu       sync.Mutex
    sessions map[string]*SessionUsage  // tapeName -> usage
}

type SessionUsage struct {
    TapeName       string
    ModelName      string
    PromptTokens   int
    CompletionTokens int
    TotalTokens    int
    Steps          int
    StartTime      time.Time
    Compacts       int
    EstimatedCost  float64  // 基于模型定价
}

// Record 在每次 LLM 调用后更新
func (t *UsageTracker) Record(tapeName, model string, usage openai.Usage) {
    t.mu.Lock()
    defer t.mu.Unlock()
    s := t.sessions[tapeName]
    s.PromptTokens += usage.PromptTokens
    s.CompletionTokens += usage.CompletionTokens
    s.TotalTokens += usage.TotalTokens
    s.Steps++
    s.EstimatedCost += estimateCost(model, usage)
}

// Summary 返回当前会话摘要
func (t *UsageTracker) Summary(tapeName string) string {
    s := t.sessions[tapeName]
    return fmt.Sprintf("[%dk tokens | $%.3f | %d steps]",
        s.TotalTokens/1000, s.EstimatedCost, s.Steps)
}
```

### 3. 模型定价配置

```toml
# ~/.dmr/config.toml
[[models]]
name = "claude-sonnet"
model = "claude-sonnet-4-20250514"
# 定价（per 1M tokens）
pricing_input = 3.0
pricing_output = 15.0
```

如果未配置定价，显示 token 数但不显示费用。

### 4. 持久化统计

将使用统计写入 tape 的 event entry 中，便于历史查询：

```go
// 在每次 agent run 结束后
a.tape.Append(ctx, tapeName, tape.Entry{
    Kind: "event",
    Payload: map[string]any{
        "type":              "usage_summary",
        "prompt_tokens":     usage.PromptTokens,
        "completion_tokens": usage.CompletionTokens,
        "steps":             result.Steps,
        "model":             modelName,
        "estimated_cost":    cost,
    },
})
```

### 5. 查询命令

```text
,usage              — 当前会话 token 消耗和费用
,usage.tape <name>  — 指定 tape 的累计统计
,usage.today        — 今日所有 tape 的合计
```

### 6. Compact 预警

在 agent 循环中，当 token 消耗接近 compact 阈值时提示：

```go
// 在 step 事件中
ratio := float64(currentTokens) / float64(compactThreshold)
if ratio > 0.8 {
    events <- AgentEvent{Kind: EventCompactWarning, Data: map[string]any{
        "current": currentTokens,
        "threshold": compactThreshold,
        "ratio": ratio,
    }}
}
```

## 实现步骤

- [ ] 创建 `pkg/agent/usage_tracker.go`（UsageTracker 核心）
- [ ] 在 agent loop 每次 LLM 调用后调用 `tracker.Record()`
- [ ] CLI renderer 在每步后显示进度信息（可通过 `[cli] show_usage = true` 配置）
- [ ] 在模型配置中添加 `pricing_input`/`pricing_output` 可选字段
- [ ] 添加 `,usage` 系列命令
- [ ] 在 agent run 结束后写入 usage event 到 tape
- [ ] 添加 compact 预警逻辑

## 代价与风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| token 计数不准确（流式模式下 usage 可能缺失） | 中 | 低 | OpenAI API 在流式最后一条 chunk 中返回 usage，需正确提取 |
| 定价信息过时 | 高 | 低 | 用户配置为准，不内置定价数据 |
| 进度信息干扰输出阅读 | 中 | 低 | 默认关闭，`show_usage = true` 手动开启；或使用 dim 颜色 |
| 多模型切换时统计混乱 | 低 | 低 | 按 model 分别记录，summary 合并展示 |

## 参考

- DMR `pkg/agent/loop.go` — `extractUsage()` 已有 usage 提取逻辑
- DMR `pkg/agent/preemptive_compact.go` — token 估算和阈值判断
- Claude Code CLI — 会话结束时显示 token 统计的参考
- OpenAI API — `usage` 字段规范
