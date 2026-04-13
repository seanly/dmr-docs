# P1: 外部插件无重连机制 — 崩溃后永久失效

> 外部插件通过 HashiCorp go-plugin（net/rpc）通信。插件进程崩溃后，当前 RPC 调用返回错误，但没有健康检查、自动重连或进程重启机制。在长运行的 `brain` 模式下，一个外部插件崩溃意味着其提供的所有工具永久失效，直到整个 DMR 进程重启。RPC 调用返回 nil 结果时也不报错。

## 问题

### 1. 无重连 / 重启机制

`pkg/plugin/external.go` 中的 `ExternalPlugin` 持有一个 `goplugin.Client` 和一个 `DMRPluginInterface`（RPC 代理）。如果外部插件进程崩溃（OOM、segfault、unhandled panic）：

1. 当前进行中的 `CallTool` RPC 返回一个网络错误
2. 后续所有 `CallTool` 调用也返回错误（RPC 连接已死）
3. **没有任何代码尝试检测进程状态或重新建立连接**

工具调用错误以 `ErrTool` 形式传给 LLM，LLM 会看到类似 `"call tool xxx: connection refused"` 的消息。LLM 可能会反复重试调用该工具，造成无意义的循环。

### 2. 无健康检查

虽然 `Plugin` 接口有可选的 `HealthChecker`：

```go
type HealthChecker interface {
    HealthCheck(ctx context.Context) error
}
```

但 `ExternalPlugin` 没有实现 `HealthChecker`。即使实现了，`Manager.HealthCheckAll()` 也没有被定期调用——只在显式请求时执行。

### 3. RPC 结果反序列化失败静默返回 nil

`pkg/plugin/external.go:113`:

```go
if len(resp.ResultJSON) > 0 {
    if err := json.Unmarshal(resp.ResultJSON, &result); err != nil {
        slog.Warn("unmarshal tool result", "error", err)
        // result 保持 nil，但没有返回 error
    }
}
return result, nil  // ← nil result + nil error = 工具看起来"成功"了但没有输出
```

如果 RPC 返回了格式错误的 JSON，工具"成功执行"但结果为空。

### 4. brain 模式的影响

`brain` 模式是长运行服务（launchd/systemd），可能运行数天。外部插件（如 Feishu bot、Aliyun CLI）是通过 `brain` 提供能力的关键组件。一次崩溃就永久丢失能力，等同于整个 channel 失效。

`brain` 支持 SIGHUP 热重载，但热重载会**重建整个插件栈**，不适合用于单个插件恢复。

## 设计方案

### 1. 进程存活检测

```go
// pkg/plugin/external.go

func (p *ExternalPlugin) isAlive() bool {
    if p.client.Exited() {
        return false
    }
    // ping RPC
    return p.rpc.Ping() == nil  // 需要在 DMRPluginInterface 中添加 Ping 方法
}
```

### 2. 自动重启策略

```go
type RestartPolicy struct {
    MaxRestarts    int           // 最大重启次数，默认 3
    RestartDelay   time.Duration // 重启间隔，默认 5s
    BackoffFactor  float64       // 退避因子，默认 2.0
    ResetAfter     time.Duration // 成功运行多久后重置重启计数，默认 5min
}

func (p *ExternalPlugin) ensureAlive() error {
    if p.isAlive() {
        return nil
    }

    p.restartMu.Lock()
    defer p.restartMu.Unlock()

    // 双检
    if p.isAlive() {
        return nil
    }

    if p.restartCount >= p.policy.MaxRestarts {
        return fmt.Errorf("plugin %s: max restarts exceeded (%d)", p.name, p.policy.MaxRestarts)
    }

    slog.Warn("restarting crashed plugin", "plugin", p.name, "attempt", p.restartCount+1)

    // 重新启动进程
    client := goplugin.NewClient(p.clientConfig)
    rpcClient, err := client.Client()
    if err != nil {
        return fmt.Errorf("restart plugin %s: %w", p.name, err)
    }

    raw, err := rpcClient.Dispense("dmr-plugin")
    if err != nil {
        client.Kill()
        return fmt.Errorf("dispense plugin %s: %w", p.name, err)
    }

    // 重新初始化
    rpc := raw.(DMRPluginInterface)
    if err := rpc.Init(&InitRequest{Config: p.initConfig}, &InitResponse{}); err != nil {
        client.Kill()
        return fmt.Errorf("reinit plugin %s: %w", p.name, err)
    }

    // 替换连接
    p.client.Kill()  // 清理旧进程
    p.client = client
    p.rpc = rpc
    p.restartCount++

    return nil
}
```

### 3. 在 CallTool 中集成存活检测

```go
func (p *ExternalPlugin) callTool(td ToolDef, ctx *tool.ToolContext, args map[string]any) (any, error) {
    // 每次调用前检查存活状态
    if err := p.ensureAlive(); err != nil {
        return nil, fmt.Errorf("plugin unavailable: %w", err)
    }

    resp := &CallToolResponse{}
    err := p.rpc.CallTool(&CallToolRequest{...}, resp)
    if err != nil {
        // 检查是否是连接错误（可重试）
        if isConnectionError(err) {
            // 标记为死亡，下次调用会触发重启
            p.markDead()
        }
        return nil, fmt.Errorf("call tool %s: %w", td.Name, err)
    }

    return p.parseResult(resp)
}
```

### 4. 修复 ResultJSON 反序列化

```go
if len(resp.ResultJSON) > 0 {
    if err := json.Unmarshal(resp.ResultJSON, &result); err != nil {
        return nil, fmt.Errorf("unmarshal tool %s result: %w", td.Name, err)  // 改为返回错误
    }
}
```

### 5. 定期健康检查（brain 模式）

```go
// cmd/dmr/brain.go — brain 主循环中添加

ticker := time.NewTicker(30 * time.Second)
defer ticker.Stop()

go func() {
    for range ticker.C {
        results := pm.HealthCheckAll(ctx)
        for name, err := range results {
            if err != nil {
                slog.Warn("plugin health check failed", "plugin", name, "error", err)
            }
        }
    }
}()
```

## 实现步骤

- [ ] 在 `DMRPluginInterface` 中添加 `Ping` 方法（或复用 `Init` 做轻量检查）
- [ ] 实现 `ExternalPlugin.isAlive()` 检测
- [ ] 实现 `RestartPolicy` 和 `ensureAlive()` 自动重启
- [ ] 在 `callTool` 中集成存活检测和连接错误识别
- [ ] 修复 `ResultJSON` 反序列化失败不返回错误的 bug
- [ ] 在 `brain` 模式添加定期健康检查循环
- [ ] `ExternalPlugin` 实现 `HealthChecker` 接口
- [ ] 保存 `clientConfig` 和 `initConfig` 以支持重启
- [ ] 添加测试：模拟插件进程崩溃后的重启和恢复

## 代价与风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| 重启期间工具短暂不可用 | 确定 | 低 | 重启通常 < 1s；LLM 看到临时错误后会重试 |
| 插件反复崩溃导致密集重启 | 中 | 中 | MaxRestarts 限制 + 指数退避 + 告警日志 |
| 重启后插件状态丢失 | 中 | 中 | 外部插件应设计为无状态或可从 tape 恢复状态 |
| Ping 增加 RPC 协议版本不兼容风险 | 低 | 中 | 旧插件不实现 Ping 时 fallback 到 `client.Exited()` |

## 参考

- `pkg/plugin/external.go:104-106` — CallTool 错误处理
- `pkg/plugin/external.go:113` — ResultJSON 反序列化静默失败
- `pkg/plugin/external.go:124-142` — Shutdown 超时 + 强杀（已正确）
- `pkg/plugin/loader.go` — 外部插件加载
- `pkg/plugin/proto/rpc.go` — RPC 协议实现
- `cmd/dmr/brain.go` — brain 长运行模式
