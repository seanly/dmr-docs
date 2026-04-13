# P2: Brain Channel 管理 — 多通道可见性与控制

## 问题

`dmr brain` 是一个多 channel 的 AI agent orchestrator，通过 webserver 插件提供 HTTP API，支持同时服务多个 tape/channel。但当前用户缺乏对运行中 channel 的可见性和控制能力：

1. **不知道有哪些活跃 channel**：brain 在后台运行，用户无法查看当前有哪些 tape 在活跃、各自在做什么
2. **无法查看 channel 状态**：某个 channel 是否在等待审批、是否卡住、消耗了多少 token
3. **无法从 CLI 管理 brain**：`dmr brain service status` 只显示进程是否运行，不显示业务状态
4. **pending approval 不可见**：如果有工具调用在等待审批，用户可能完全不知道，导致任务长时间挂起
5. **无法从外部中断某个 channel**：如果某个 tape 上的 agent 循环失控，只能杀掉整个 brain 进程

### 现状

- `plugins/webserver/` 提供了 REST API（tape 列表、history、chat），但没有聚合的 dashboard 视图
- `dmr brain service status` 只检查进程状态（launchd/systemd level）
- 没有 CLI 命令连接到运行中的 brain 查询内部状态

## 设计方案

### 1. Brain Status API

在 webserver 插件中添加聚合状态端点：

```go
// plugins/webserver/handlers.go

// GET /api/brain/status
type BrainStatus struct {
    Uptime      time.Duration      `json:"uptime"`
    ActiveTapes []TapeStatus       `json:"active_tapes"`
    Pending     []PendingApproval  `json:"pending_approvals"`
    TotalTokens int                `json:"total_tokens"`
    Version     string             `json:"version"`
}

type TapeStatus struct {
    Name        string        `json:"name"`
    State       string        `json:"state"`       // idle / running / awaiting_approval
    LastActivity time.Time    `json:"last_activity"`
    Steps       int           `json:"steps"`
    TokensUsed  int           `json:"tokens_used"`
}

type PendingApproval struct {
    TapeName  string         `json:"tape_name"`
    ToolName  string         `json:"tool_name"`
    Args      map[string]any `json:"args"`
    WaitingSince time.Time   `json:"waiting_since"`
}
```

### 2. CLI 查询命令

新增 `dmr brain status` 子命令，连接到运行中的 brain HTTP API：

```go
// cmd/dmr/brain.go

// dmr brain status — 查询 brain 内部状态
func brainStatusCmd() *cobra.Command {
    return &cobra.Command{
        Use:   "status",
        Short: "Show brain internal status (channels, pending approvals, usage)",
        RunE: func(cmd *cobra.Command, args []string) error {
            // 连接到 brain 的 HTTP API
            resp, err := http.Get(fmt.Sprintf("http://localhost:%d/api/brain/status", port))
            // 格式化输出
            printBrainStatus(status)
            return nil
        },
    }
}
```

输出示例：

```text
DMR Brain — up 3h 42m | v0.3.0

ACTIVE CHANNELS (3):
  cli:main       idle        last: 2m ago     1.2k tokens
  web:deploy     running     step 3/20        4.8k tokens
  web:review     approval    waiting 5m       2.1k tokens

PENDING APPROVALS (1):
  web:review  shell: kubectl delete pod nginx-xxx
              waiting since 14:23 (5m ago)
              → approve at http://localhost:8080/approve/web:review

TOTAL: 8.1k tokens | 3 channels
```

### 3. Channel 控制

```text
dmr brain cancel <tape>    — 取消指定 channel 上正在运行的 agent 循环
dmr brain approve <tape>   — 从 CLI 批准 pending approval（无需打开 web）
dmr brain deny <tape>      — 从 CLI 拒绝 pending approval
```

### 4. Approval 通知

当 brain 有 pending approval 超过可配置阈值时，主动通知：

```toml
# ~/.dmr/config.toml
[brain]
approval_timeout_warn = "5m"   # pending 超过 5 分钟时告警
notification = "desktop"       # desktop / webhook / none
```

对于 macOS，可使用 `osascript` 发送通知；对于 Linux，使用 `notify-send`：

```go
func sendDesktopNotification(title, body string) {
    switch runtime.GOOS {
    case "darwin":
        exec.Command("osascript", "-e",
            fmt.Sprintf(`display notification "%s" with title "%s"`, body, title)).Run()
    case "linux":
        exec.Command("notify-send", title, body).Run()
    }
}
```

## 实现步骤

- [ ] 在 webserver 插件中实现 `/api/brain/status` 端点
- [ ] 收集 tape 活跃状态（需要 agent 层暴露运行状态）
- [ ] 收集 pending approval 状态（需要 approver 层暴露等待队列）
- [ ] 实现 `dmr brain status` CLI 命令
- [ ] 实现 `dmr brain cancel/approve/deny` 控制命令
- [ ] 添加 approval 超时通知（可选 desktop/webhook）
- [ ] 在 web UI 中添加 dashboard 视图（可延后）

## 代价与风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| status API 暴露敏感信息（工具参数） | 中 | 中 | API 需要 auth；args 只显示摘要，不显示完整值 |
| cancel 导致 tape 状态不一致 | 中 | 中 | cancel 通过 context 取消实现，确保 tape 写入最后状态 |
| desktop 通知在 headless 环境失败 | 低 | 低 | notification = "none" 默认值；headless 时自动降级 |
| brain 未启动时 CLI status 报错 | 确定 | 低 | 友好错误提示 "brain is not running, use `dmr brain service start`" |

## 参考

- DMR `plugins/webserver/` — 当前 HTTP API 实现
- DMR `cmd/dmr/brain.go` — brain 命令和 service 子命令
- DMR `plugins/webserver/approver.go` — WebApproverAdapter
- k9s — Kubernetes CLI dashboard 的交互参考
