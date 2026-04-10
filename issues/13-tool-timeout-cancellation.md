# P1: 工具执行无超时 + 无取消

> 灵感来源：TapeAgents 同样缺少此能力，但 TapeAgents 的 `FatalError` 分级模式值得借鉴

## 问题

DMR 的工具执行没有超时机制，也无法取消。

`pkg/tool/executor.go` 中所有工具调用：

```go
out, err := rc.tool.Handler(ctx, rc.args)
```

这里的 `ctx` 是 `*ToolContext`，**不是 `context.Context`**。工具 Handler 签名：

```go
Handler func(ctx *ToolContext, args map[string]any) (any, error)
```

这意味着：
1. **无法超时**：一个挂起的 shell 命令、网络请求、MCP 调用会阻塞整个 agent 循环永远
2. **无法取消**：用户按 Ctrl+C 或 Web 客户端断开连接后，工具仍在后台运行
3. **并行 subagent 中一个挂起会阻塞所有** — `wg.Wait()` 等待所有 goroutine 完成

同时，OPA 策略检查也使用 `context.Background()`（`executor.go:431,449`），忽略调用者的取消信号。

**TapeAgents 同样没有工具超时**（`environment.py` 中 `tool.run(action)` 无超时），但 TapeAgents 是研究框架，DMR 面向生产环境，这个缺陷更严重。

## 修复方案

### 1. ToolContext 嵌入 context.Context

```go
// pkg/tool/context.go

type ToolContext struct {
    context.Context              // 嵌入标准 context，支持超时和取消
    Tape    string
    RunID   string
    State   map[string]any
    // ...
}
```

### 2. 工具执行加超时包装

```go
// pkg/tool/executor.go

func (e *ToolExecutor) executeWithTimeout(
    ctx *ToolContext, tool *Tool, args map[string]any, timeout time.Duration,
) (any, error) {
    if timeout <= 0 {
        return tool.Handler(ctx, args)
    }
    
    timeoutCtx, cancel := context.WithTimeout(ctx, timeout)
    defer cancel()
    
    childCtx := ctx.WithContext(timeoutCtx)
    
    type result struct {
        out any
        err error
    }
    ch := make(chan result, 1)
    go func() {
        out, err := tool.Handler(childCtx, args)
        ch <- result{out, err}
    }()
    
    select {
    case r := <-ch:
        return r.out, r.err
    case <-timeoutCtx.Done():
        return nil, fmt.Errorf("tool %q timed out after %v", tool.Name, timeout)
    }
}
```

### 3. 配置

```toml
[agent]
tool_timeout = "120s"        # 默认工具超时
subagent_timeout = "300s"    # subagent 超时（允许更长）

# 每个工具可覆盖
[tools.shell]
timeout = "60s"

[tools.webFetch]
timeout = "30s"
```

### 4. 工具级超时声明

```go
type Tool struct {
    Name        string
    Description string
    Handler     func(ctx *ToolContext, args map[string]any) (any, error)
    Timeout     time.Duration  // 新增：工具级默认超时
    // ...
}
```

## 影响

- `pkg/tool/context.go` — ToolContext 增加 Context 嵌入
- `pkg/tool/tool.go` — Tool 增加 Timeout 字段
- `pkg/tool/executor.go` — 执行逻辑包装超时
- 所有内置工具的 Handler 需要检查 `ctx.Done()`（渐进式改造）

## 参考

- TapeAgents 的缺失：`tapeagents/environment.py` 无超时
- TapeAgents FatalError 模式：`tapeagents/environment.py:148-149`（区分可恢复和不可恢复错误）
- DMR executor：`pkg/tool/executor.go:137-189`
