# P1: 流式输出 — CLI 实时显示 LLM 生成过程

## 问题

当前 DMR 的 `Agent.Run()` 是一次性返回结果的（`pkg/agent/loop.go`），CLI 的 `session.go` 在 LLM 完成全部推理后才调用 `renderer.PrintAssistant(result.Output)` 打印输出。

对于长时间思考的 LLM 调用（尤其是带 reasoning 的模型），用户体验极差：

1. **黑洞等待**：用户提交 prompt 后，几秒到几十秒内无任何输出，无法判断是在处理还是卡死
2. **无法提前中断**：看到前几句就知道模型方向错了，但只能等完整输出后再重试
3. **工具调用不透明**：多步 agent 循环中，用户不知道当前在执行哪个工具、执行到第几步
4. **长输出耐心成本高**：对于生成代码、分析报告等场景，等待时间直接影响使用意愿

### 现状分析

- `pkg/openai/client.go` 已有 `CreateChatCompletionStream()` 方法，底层支持流式
- `pkg/client/chat.go` 的 `ChatClient` 同时支持 `Chat()` 和 `ChatStream()` 方法
- 但 `pkg/agent/loop.go` 只使用了 `Chat()`（非流式），stream 能力未接入 agent 循环
- `plugins/cli/renderer.go` 的输出方法全部是一次性打印，没有增量渲染接口

## 设计方案

### 方案概览

```
Agent Loop  ──stream──>  EventChannel  ──>  CLI Renderer (逐 token 打印)
                                       ──>  WebServer (SSE push)
                                       ──>  Tape (完整记录)
```

### 1. Agent 事件流

```go
// pkg/agent/events.go

type EventKind string

const (
    EventToken       EventKind = "token"         // LLM 输出的单个 token
    EventToolStart   EventKind = "tool_start"    // 工具开始执行
    EventToolEnd     EventKind = "tool_end"      // 工具执行完成
    EventStepStart   EventKind = "step_start"    // agent 循环新一步
    EventCompact     EventKind = "compact"       // compact 触发
    EventThinking    EventKind = "thinking"      // reasoning/thinking 内容
)

type AgentEvent struct {
    Kind    EventKind
    Data    map[string]any  // token: {"text": "..."}, tool_start: {"name": "shell", "args": {...}}
    Step    int             // 当前步数
    MaxStep int             // 最大步数
}
```

### 2. 流式 Run 方法

```go
// pkg/agent/loop.go

// RunStream 返回事件 channel，调用方可逐事件消费
func (a *Agent) RunStream(ctx context.Context, tapeName, prompt string, historyAfterEntryID int32) (<-chan AgentEvent, <-chan *Result, <-chan error) {
    events := make(chan AgentEvent, 64)
    resultCh := make(chan *Result, 1)
    errCh := make(chan error, 1)
    
    go func() {
        defer close(events)
        defer close(resultCh)
        defer close(errCh)
        
        result, err := a.runStreaming(ctx, tapeName, prompt, historyAfterEntryID, events)
        if err != nil {
            errCh <- err
            return
        }
        resultCh <- result
    }()
    
    return events, resultCh, errCh
}
```

### 3. CLI 增量渲染

```go
// plugins/cli/renderer.go

func (r *Renderer) StreamAssistant(events <-chan agent.AgentEvent) string {
    r.assistant.Print("Assistant> ")
    var fullText strings.Builder
    
    for event := range events {
        switch event.Kind {
        case agent.EventToken:
            text := event.Data["text"].(string)
            fmt.Print(text)  // 逐 token 打印
            fullText.WriteString(text)
        case agent.EventToolStart:
            fmt.Println()  // 换行
            r.PrintToolCall(event.Data["name"].(string), "", "")
        case agent.EventStepStart:
            r.system.Printf("\n  [step %d/%d]\n", event.Step, event.MaxStep)
        case agent.EventThinking:
            r.toolArgs.Printf("  (thinking...)\n")
        }
    }
    fmt.Println()
    return fullText.String()
}
```

### 4. 兼容性

- `Run()` 保持不变，内部可以复用 `RunStream()` 收集完整结果
- webserver 插件可通过相同的 event channel 推送 SSE
- Tape 记录不受影响——最终完整文本仍写入 tape

## 实现步骤

- [ ] 定义 `AgentEvent` 类型和事件常量（`pkg/agent/events.go`）
- [ ] 在 agent loop 内部将 `Chat()` 切换为 `ChatStream()`，发送 token 事件
- [ ] 为工具执行阶段添加 `tool_start`/`tool_end` 事件
- [ ] 实现 `RunStream()` 方法，保持 `Run()` 向后兼容
- [ ] 改造 CLI `session.go` 使用流式渲染
- [ ] webserver 插件对接事件流（可延后）
- [ ] 添加 `--no-stream` flag 或配置项，允许禁用流式输出

## 代价与风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| 流式模式下 tool_calls 解析时机变化 | 中 | 高 | OpenAI stream 的 function_call 是逐块传来的，需要积累完整 JSON 后再解析 |
| token 事件高频导致 channel 背压 | 低 | 中 | 使用带缓冲的 channel（64），消费方丢弃不影响正确性 |
| 流式与非流式结果不一致 | 低 | 中 | 用相同的 prompt/temperature 对比验证 |
| 中断（Ctrl+C）时的资源清理 | 中 | 中 | context 取消传播 + 确保 tape 写入完整的部分结果 |

## 参考

- DMR `pkg/openai/client.go` — 已有 `CreateChatCompletionStream()` 实现
- DMR `pkg/client/chat.go` — `ChatStream()` 方法
- DMR `plugins/cli/session.go` — 当前非流式调用路径
- Claude Code CLI — 流式输出 + 工具调用状态的参考体验
