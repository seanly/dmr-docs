# P1: PermissionRequest Hook 化 — 解耦审批后端

> 当前审批流程硬编码在 OPA 插件的 `handleApproval` 方法中，通过 `ProvideApprover` hook 间接对接审批后端。Claude Code 将审批请求作为独立的 `PermissionRequest` 事件暴露，任何 hook 都可以拦截并自动审批或拒绝。DMR 应将审批流程从 OPA 插件中解耦，成为独立的 hook 事件。

## 问题

### 1. 审批逻辑与策略引擎耦合

当前审批流程：

```text
OPA 评估 → require_approval
  → OPA 插件 handleApproval()
    → 检查 ApprovalCache
    → 调用 ProvideApprover hook 获取 Approver
    → 调用 approver.RequestApproval()
    → 缓存结果
```

`handleApproval` 在 `plugins/opapolicy/opapolicy.go` 中实现，包含缓存检查、审批人查找、结果持久化等逻辑。这意味着：

- 如果要替换 OPA 为其他策略引擎，审批逻辑也需要重新实现
- 审批缓存（`cache.go`）与 OPA 插件紧耦合
- 其他插件无法在审批流程中插入逻辑（如审计、通知）

### 2. ProvideApprover 是间接机制

`ProvideApprover` hook 只负责"谁来审批"，不负责"是否要审批"。其他插件无法：

- 自动审批特定模式的请求（如 CI 环境中自动 approve 所有读操作）
- 在审批前注入额外信息（如给审批人展示历史决策统计）
- 在审批后触发通知（如发送飞书消息告知团队）

### 3. 批量审批体验不一致

`batchBeforeToolCallTyped` 中的批量审批逻辑与单次审批逻辑重复实现，且批量审批的 UX（合并显示多个待审批项）由 OPA 插件控制，其他插件无法定制。

## 设计方案

### 1. 新增 PermissionRequest Hook 事件

```go
// pkg/plugin/typed_hooks.go

// PermissionRequestArgs is passed to PermissionRequest hook handlers.
type PermissionRequestArgs struct {
    Tool     any            // *tool.Tool
    Args     map[string]any // tool call arguments
    ToolCtx  any            // *tool.ToolContext
    Decision Decision       // the decision that triggered the approval request
}

// PermissionRequestResult is the return value from PermissionRequest hooks.
type PermissionRequestResult struct {
    // Action overrides the approval flow.
    // "allow" — auto-approve, skip interactive prompt
    // "deny"  — auto-deny, skip interactive prompt
    // ""      — defer to normal approval flow
    Action string

    // Reason explains the auto-decision (shown to user and logged).
    Reason string

    // Persistence controls how the decision is cached.
    // "once" / "session" / "always" / "" (no caching)
    Persistence string
}

// PermissionRequestFunc is the typed signature.
type PermissionRequestFunc func(ctx context.Context, args PermissionRequestArgs) (*PermissionRequestResult, error)
```

### 2. 从 OPA 插件中提取审批流程

```go
// pkg/tool/executor.go — 审批流程移到 ToolExecutor 层

func (e *ToolExecutor) handleApproval(ctx context.Context, t *tool.Tool, args map[string]any, decision Decision) error {
    // Step 1: 触发 PermissionRequest hook
    result, err := e.hooks.CallPermissionRequest(ctx, PermissionRequestArgs{
        Tool: t, Args: args, Decision: decision,
    })
    if err != nil {
        return err
    }

    // Step 2: hook 已做出决策
    if result != nil && result.Action != "" {
        switch result.Action {
        case "allow":
            return nil
        case "deny":
            return fmt.Errorf("denied by permission hook: %s", result.Reason)
        }
    }

    // Step 3: 没有 hook 拦截，走正常审批流程
    approver := e.findApprover(ctx)
    choice, err := approver.RequestApproval(ctx, approvalRequest)
    // ...
}
```

### 3. OPA 插件简化

OPA 插件不再处理审批流程，只负责策略评估和返回 Decision：

```go
// plugins/opapolicy/opapolicy.go — 简化后

func (p *OPAPolicyPlugin) beforeToolCallTyped(ctx context.Context, args plugin.BeforeToolCallArgs) (*plugin.BeforeToolCallResult, error) {
    decision, err := p.evaluate(ctx, input)
    if err != nil {
        return nil, err
    }
    switch decision.Action {
    case "allow":
        return nil, nil
    case "deny":
        return nil, fmt.Errorf("denied: %s", decision.Reason)
    case "require_approval":
        // 不再自己处理审批，返回特殊错误让 ToolExecutor 处理
        return nil, &plugin.ApprovalRequiredError{Decision: decision}
    }
    return nil, nil
}
```

### 4. 审批缓存独立化

将 `ApprovalCache` 从 OPA 插件中提取为独立组件，注册为 PermissionRequest hook：

```go
// pkg/approval/cache_hook.go

type CacheHook struct {
    cache *ApprovalCache
}

func (h *CacheHook) HandlePermissionRequest(ctx context.Context, args PermissionRequestArgs) (*PermissionRequestResult, error) {
    if cached := h.cache.Check(args.Tool, args.Args); cached != nil {
        return &PermissionRequestResult{
            Action: "allow",
            Reason: fmt.Sprintf("cached approval: %s", cached.Reason),
        }, nil
    }
    return nil, nil // defer to normal flow
}
```

### 5. 示例：CI 环境自动审批插件

```go
// plugins/ci_auto_approve/plugin.go

func (p *CIAutoApprovePlugin) HandlePermissionRequest(ctx context.Context, args PermissionRequestArgs) (*PermissionRequestResult, error) {
    if os.Getenv("CI") != "true" {
        return nil, nil // 非 CI 环境，不干预
    }
    if args.Decision.Risk == "critical" {
        return &PermissionRequestResult{
            Action: "deny",
            Reason: "critical-risk operations are denied in CI",
        }, nil
    }
    return &PermissionRequestResult{
        Action: "allow",
        Reason: "auto-approved in CI environment",
    }, nil
}
```

## 实现步骤

- [ ] 定义 `PermissionRequestArgs`、`PermissionRequestResult`、`PermissionRequestFunc`（`pkg/plugin/typed_hooks.go`）
- [ ] 在 `HookRegistry` 中注册 `PermissionRequest` 事件
- [ ] 定义 `ApprovalRequiredError` 类型（`pkg/plugin/approval.go`）
- [ ] 将审批流程从 `plugins/opapolicy/opapolicy.go` 移到 `pkg/tool/executor.go`
- [ ] 将 `ApprovalCache` 提取为独立包（`pkg/approval/`）
- [ ] 将 `ApprovalCache` 注册为 PermissionRequest hook
- [ ] 简化 OPA 插件：只返回 Decision，不处理审批
- [ ] 同步修改 `batchBeforeToolCallTyped` 的批量审批逻辑
- [ ] 迁移测试
- [ ] 更新插件开发文档：如何编写 PermissionRequest hook

## 代价与风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| 审批流程拆分后行为不一致 | 中 | 高 | 充分测试：单次审批、批量审批、缓存命中、缓存失效 |
| 多 PermissionRequest hook 的优先级冲突 | 中 | 中 | 与 BeforeToolCall 一致：deny > allow > defer；最严格决策优先 |
| 现有 Approver 实现（CLI、Web、飞书）需要适配 | 确定 | 低 | Approver 接口不变，只是调用位置从 OPA 插件移到 ToolExecutor |
| CI 自动审批被滥用绕过安全策略 | 中 | 高 | CI 自动审批插件仍受 managed deny 规则约束；critical-risk 默认拒绝 |

## 参考

- Claude Code `PermissionRequest` 事件 — auto-approve/deny + `updatedPermissions` 持久化
- `plugins/opapolicy/opapolicy.go:141` — 当前 handleApproval 实现
- `plugins/opapolicy/cache.go` — ApprovalCache
- `pkg/plugin/approval.go` — Decision、Approver、TapeAwareApprover 接口
- `pkg/tool/executor.go:415` — batchPolicyCheck
