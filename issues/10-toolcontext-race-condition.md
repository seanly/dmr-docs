# P0: ToolContext 并发数据竞争

> 灵感来源：TapeAgents 的不可变 tape 传递模式（值语义消除竞争）

## 问题

并行 subagent 执行时存在数据竞争。

`pkg/tool/executor.go:228-231`：

```go
go func(i int, rc resolvedCall) {
    defer wg.Done()
    out, err := rc.tool.Handler(ctx, rc.args)  // ctx 是 *ToolContext，共享
    resultCh <- parallelResult{index: i, out: out, err: err}
}(idx, rc)
```

`ToolContext.State` 是 `map[string]any`，多个 goroutine 共享同一个 map。Go 的 map **不是并发安全的**。如果任何 subagent handler 读写 `ctx.State`（而 `_runtime_agent`、`_tape_store` 等确实存在于其中），会产生 data race。

更危险的是 `Agent` 本身也被共享 — `subagentHandler` 从 `ctx.State["_runtime_agent"]` 取出 `*agent.Agent`，多个 goroutine 在同一个 Agent 实例上并发调用 `RunSubagent()`。虽然 `sync.Map` 字段本身是安全的，但复合操作不是原子的。

## TapeAgents 如何避免

TapeAgents 没有并发执行问题，因为：
1. Tape 是不可变值对象 — 每个 agent 拿到的是独立副本
2. Agent.run() 是顺序执行的生成器，不涉及并发
3. 环境 react() 也是同步的

这不是 TapeAgents "做对了"什么，而是 DMR 在引入并行能力时没有完全处理好共享状态。

## 修复方案

### 方案 A：每个 goroutine 独立 ToolContext（推荐）

```go
func (e *ToolExecutor) executeWithParallelSubagents(...) {
    for _, idx := range subagentIndices {
        rc := resolved[idx]
        
        // 为每个并行调用创建独立的 ToolContext
        childCtx := ctx.Clone()  // 深拷贝 State map
        
        wg.Add(1)
        go func(i int, rc resolvedCall, c *ToolContext) {
            defer wg.Done()
            out, err := rc.tool.Handler(c, rc.args)
            resultCh <- parallelResult{index: i, out: out, err: err}
        }(idx, rc, childCtx)
    }
}
```

```go
// pkg/tool/context.go 新增
func (c *ToolContext) Clone() *ToolContext {
    newState := make(map[string]any, len(c.State))
    for k, v := range c.State {
        newState[k] = v  // 浅拷贝值（Agent 指针共享是 OK 的，因为有 sync.Map）
    }
    return &ToolContext{
        Tape:    c.Tape,
        RunID:   c.RunID,
        State:   newState,
        Context: c.Context,
        // ... 其他字段
    }
}
```

### 方案 B：State 换成 sync.Map

```go
type ToolContext struct {
    State sync.Map  // 替代 map[string]any
}
```

侵入性更大，所有 `ctx.State["key"]` 都要改为 `ctx.State.Load("key")`。

**推荐方案 A** — 改动最小，语义最清晰。

## 验证

```bash
# 用 race detector 跑测试
go test -race ./pkg/tool/ ./pkg/agent/
```

## 参考

- DMR executor 并行代码：`pkg/tool/executor.go:192-303`
- DMR ToolContext 定义：`pkg/tool/context.go`
