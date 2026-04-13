# P0: BeforeToolCall 参数改写与上下文注入 — 借鉴 Claude Code Hook 能力

> 当前 `BeforeToolCall` 只能返回 `error`（拦截）或 `nil`（放行），无法修改工具参数或向 LLM 注入上下文。Claude Code 的 PreToolUse hook 支持 `updatedInput`（参数改写）和 `additionalContext`（上下文注入），让策略从"硬拦截"进化到"软引导"。

## 问题

### 1. 参数改写：只能拦不能改

现实场景中，很多工具调用不需要拦截，而是需要**纠正**：

| 场景 | 当前行为 | 期望行为 |
|------|---------|---------|
| `git push --force` | deny 或 require_approval | 自动改写为 `git push --force-with-lease` |
| `rm -rf ./build` | require_approval | 自动加上 `--interactive` 或注入安全前缀 |
| shell 命令缺少 `set -e` | 放行 | 自动注入 `set -euo pipefail` 前缀 |
| 写文件路径包含 `..` | deny | 自动规范化为绝对路径 |

当前 `BeforeToolCallFunc` 签名：

```go
// pkg/plugin/typed_hooks.go:92
type BeforeToolCallFunc func(ctx context.Context, args BeforeToolCallArgs) error
```

只有 `error` 返回值，无法传递修改后的参数。

### 2. 上下文注入：策略无法"教"LLM

OPA 策略只能做门控决策（allow/deny/require_approval），无法向 LLM 注入提示信息。例如：

- deny 了 `kubectl delete` 后，无法告知 LLM "请先用 `kubectl get` 确认目标 pod"
- allow 了 `git commit` 后，无法提醒 LLM "这个仓库要求 conventional commits 格式"
- require_approval 时，无法给审批人附加风险说明

Claude Code 的 `additionalContext` 机制让 hook 返回的文本作为 system reminder 注入 LLM 对话，实现软引导。

## 设计方案

### 1. 扩展 BeforeToolCall 返回值

新增 `BeforeToolCallResult` 结构体，替代裸 `error` 返回：

```go
// pkg/plugin/approval.go

// BeforeToolCallResult extends the hook return value beyond simple allow/deny.
type BeforeToolCallResult struct {
    // UpdatedArgs replaces the original tool call arguments.
    // nil means no modification (keep original args).
    UpdatedArgs map[string]any

    // Context is injected into the LLM conversation as a system reminder.
    // Empty string means no injection.
    Context string
}
```

### 2. 修改 BeforeToolCallFunc 签名

```go
// pkg/plugin/typed_hooks.go

// BeforeToolCallFunc is the typed signature for BeforeToolCall hook handlers.
// Returns:
//   - (nil, nil): allow, no modifications
//   - (result, nil): allow with modifications (updated args and/or context)
//   - (_, error): deny (error is the reason)
type BeforeToolCallFunc func(ctx context.Context, args BeforeToolCallArgs) (*BeforeToolCallResult, error)
```

### 3. HookRegistry 合并多个 hook 的结果

```go
// pkg/plugin/hooks.go

func (r *HookRegistry) CallBeforeToolCall(ctx context.Context, args BeforeToolCallArgs) (*BeforeToolCallResult, error) {
    var merged BeforeToolCallResult
    for _, h := range r.beforeToolCall {
        result, err := h.Handler(ctx, args)
        if err != nil {
            return nil, err // 任何 hook deny 即停止
        }
        if result != nil {
            // 参数改写：后注册的覆盖先注册的（按 priority 排序）
            if result.UpdatedArgs != nil {
                if merged.UpdatedArgs == nil {
                    merged.UpdatedArgs = make(map[string]any)
                }
                for k, v := range result.UpdatedArgs {
                    merged.UpdatedArgs[k] = v
                }
            }
            // 上下文注入：拼接所有 hook 的 context
            if result.Context != "" {
                if merged.Context != "" {
                    merged.Context += "\n"
                }
                merged.Context += result.Context
            }
        }
    }
    return &merged, nil
}
```

### 4. ToolExecutor 应用结果

```go
// pkg/tool/executor.go — Execute 方法

result, err := pm.Hooks().CallBeforeToolCall(ctx, hookArgs)
if err != nil {
    return nil, err // denied
}
if result != nil {
    if result.UpdatedArgs != nil {
        args = result.UpdatedArgs // 替换参数
    }
    if result.Context != "" {
        // 注入到工具结果的附加上下文中
        toolCtx.AdditionalContext = append(toolCtx.AdditionalContext, result.Context)
    }
}
```

### 5. OPA 策略扩展

Decision 结构体已有 `Details map[string]any`，可直接利用：

```rego
# plugins/opapolicy/policies/default.rego

# 自动将 git push --force 改写为 --force-with-lease
decision := {
    "action": "modify",
    "reason": "replaced --force with --force-with-lease for safety",
    "risk": "medium",
    "updated_args": {"command": replace(input.args.command, "--force", "--force-with-lease")},
    "context": "This repository requires --force-with-lease instead of --force."
} if {
    input.tool == "shell"
    contains(input.args.command, "git push")
    contains(input.args.command, "--force")
    not contains(input.args.command, "--force-with-lease")
}
```

OPA 插件解析 decision 时提取 `updated_args` 和 `context`：

```go
// plugins/opapolicy/opapolicy.go

case "allow", "modify":
    result := &plugin.BeforeToolCallResult{}
    if updatedArgs, ok := decision.Details["updated_args"].(map[string]any); ok {
        result.UpdatedArgs = updatedArgs
    }
    if ctx, ok := decision.Details["context"].(string); ok {
        result.Context = ctx
    }
    return result, nil
```

### 6. BatchBeforeToolCall 同步扩展

`BatchBeforeToolCallFunc` 也需要返回对应的 `[]*BeforeToolCallResult`，与单次调用保持一致。

## 实现步骤

- [ ] 定义 `BeforeToolCallResult` 结构体（`pkg/plugin/approval.go`）
- [ ] 修改 `BeforeToolCallFunc` 签名（`pkg/plugin/typed_hooks.go`）
- [ ] 修改 `HookRegistry.CallBeforeToolCall` 合并逻辑（`pkg/plugin/hooks.go`）
- [ ] 同步修改 `BatchBeforeToolCallFunc` 和 `CallBatchBeforeToolCall`
- [ ] 修改 `ToolExecutor.Execute` 应用 `UpdatedArgs` 和 `Context`（`pkg/tool/executor.go`）
- [ ] 修改 OPA 插件解析 `updated_args` 和 `context`（`plugins/opapolicy/opapolicy.go`）
- [ ] 在 default.rego 中添加示例改写规则
- [ ] 修改所有现有 `BeforeToolCallFunc` 实现适配新签名
- [ ] 添加单元测试：参数改写、上下文注入、多 hook 合并
- [ ] 更新插件开发文档

## 代价与风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| 参数改写引入安全漏洞（改写逻辑有 bug） | 中 | 高 | 改写后仍需经过 OPA 二次评估（可选配置） |
| 多 hook 参数改写冲突 | 低 | 中 | 按 priority 顺序应用；审计日志记录每次改写 |
| 上下文注入被用于 prompt injection | 低 | 中 | context 标记为 system reminder，非用户输入 |
| 破坏所有现有 BeforeToolCallFunc 实现 | 确定 | 中 | 编译器会捕获所有签名不匹配；统一修改工作量可控 |

## 参考

- Claude Code `PreToolUse` hook — `updatedInput` 和 `additionalContext` 机制
- `pkg/plugin/typed_hooks.go:85-94` — 当前 BeforeToolCallArgs 定义
- `pkg/plugin/typed_hooks.go:92` — 当前 BeforeToolCallFunc 签名
- `pkg/plugin/approval.go` — Decision 结构体
- `pkg/tool/executor.go` — ToolExecutor.Execute 流程
- `plugins/opapolicy/opapolicy.go:97-138` — OPA 插件 beforeToolCallTyped
