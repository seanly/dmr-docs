# P1: Hook 系统类型安全不完整 — any 参数与 typed wrapper 覆盖缺口

> Hook 系统的基础签名是 `func(ctx, ...any) (any, error)`，typed wrappers 仅覆盖 7/13 个 hook 且自身仍含 5 个 `any` 字段。部分 `any` 标注为"避免循环导入"但实际已无循环依赖。类型断言失败被静默丢弃（`_, _ =` 模式），安全关键路径上缺乏编译时保障。

## 问题

### 1. 基础签名完全无类型

`pkg/plugin/hooks.go:62`:

```go
type HookFunc func(ctx context.Context, args ...any) (any, error)
```

所有 hook 注册和调度都经过此签名。参数个数、类型完全依赖运行时约定。

### 2. Typed Wrappers 覆盖不完整

`pkg/plugin/typed_hooks.go` 为部分 hook 提供了类型化包装，但覆盖率不足：

| Hook | 有 typed wrapper？ |
|------|--------------------|
| `RegisterCoreTools` | 否（仅有便捷注册方法，调度仍为 `HookFunc`） |
| `RegisterExtendedTools` | 否 |
| `SystemPrompt` | 否 |
| `BeforeToolCall` | **是** |
| `BatchBeforeToolCall` | **是** |
| `ProvideApprover` | 否 |
| `ProvideChatInterface` | 否 |
| `InterceptInput` | **是** |
| `SetAgentRunner` | **是** |
| `SetTapeReader` | **是** |
| `SetTapeControl` | **是** |
| `SetHostService` | **是** |
| `AfterAgentRun` | **是** |
| `DiscoveredToolsCleared` | 否 |

**5-6 个 hook 完全依赖无类型调度**，包括安全关键的 `ProvideApprover`。

### 3. Typed Wrappers 自身含 any 字段

即使有 typed wrapper 的 hook，其参数结构体仍包含 `any` 字段：

```go
// pkg/plugin/typed_hooks.go

type InterceptInputArgs struct {
    TapeStore  any // tape.TapeStore — uses any to avoid circular import
    TapeManager any // *tape.TapeManager
}

type BeforeToolCallArgs struct {
    Tool    any // *tool.Tool — uses any to avoid circular import
    Args    map[string]any
    ToolCtx any // *tool.ToolContext
}

type BatchBeforeToolCallArgs struct {
    Items any // []tool.BatchCheckItem — uses any to avoid circular import
}
```

5 个字段使用 `any`，理由是"避免循环导入"。

### 4. "避免循环导入"理由不成立

`pkg/plugin/typed_hooks.go` 第 7 行已经 `import "github.com/seanly/dmr/pkg/tool"`。既然已经导入了 `tool` 包，`Tool any` 和 `ToolCtx any` 完全可以改为 `Tool *tool.Tool` 和 `ToolCtx *tool.ToolContext`。这是一个过时的 workaround 残留。

对于 `TapeStore` 和 `TapeManager`，如果确实存在循环导入，正确做法是将共享接口提取到 `pkg/contract` 包（见 #45），而非用 `any` 逃逸。

### 5. 断言失败被静默吞掉

消费者端的模式：

```go
// plugins/opapolicy/opapolicy.go:98-103
t, _ := args.Tool.(*tool.Tool)        // 断言失败 → t = nil
toolCtx, _ := args.ToolCtx.(*tool.ToolContext)  // 断言失败 → toolCtx = nil
if t == nil || toolCtx == nil {
    return nil  // 静默返回 nil，OPA 策略检查被跳过
}
```

这意味着：如果 `Tool` 或 `ToolCtx` 的类型在重构中改变了，OPA 策略引擎会**静默跳过所有工具调用检查**，而不是报错。对于 DMR 的安全核心来说，这是不可接受的失败模式。

## 设计方案

### 阶段 1：修复已知的 stale any

将 `typed_hooks.go` 中已导入包的 `any` 字段改为具体类型：

```go
type BeforeToolCallArgs struct {
    Tool    *tool.Tool        // 不再是 any
    Args    map[string]any
    ToolCtx *tool.ToolContext  // 不再是 any
}

type BatchBeforeToolCallArgs struct {
    Items []tool.BatchCheckItem  // 不再是 any
}
```

这是零风险变更——`pkg/plugin` 已经 import 了 `pkg/tool`。

### 阶段 2：为剩余 hook 补全 typed wrappers

为缺失的 5 个 hook 添加类型化参数和调度函数：

```go
// ProvideApprover
type ProvideApproverFunc func(ctx context.Context) (Approver, error)

func RegisterProvideApprover(r *HookRegistry, plugin string, pri int, fn ProvideApproverFunc) {
    r.RegisterHook(HookProvideApprover, plugin, pri, func(ctx context.Context, args ...any) (any, error) {
        return fn(ctx)
    })
}

// RegisterCoreTools
type RegisterToolsFunc func(ctx context.Context) ([]*tool.Tool, error)

func RegisterCoreToolsTyped(r *HookRegistry, plugin string, pri int, fn RegisterToolsFunc) {
    r.RegisterHook(HookRegisterCoreTools, plugin, pri, func(ctx context.Context, args ...any) (any, error) {
        return fn(ctx)
    })
}
```

### 阶段 3：解决 tape 包的循环导入

对于 `InterceptInputArgs` 中的 `TapeStore` 和 `TapeManager`，提取接口到无依赖的 `pkg/contract` 包：

```go
// pkg/contract/tape.go
type TapeStore interface {
    FetchAll(tape string, opts *FetchOpts) ([]TapeEntry, error)
    Append(tape string, entry TapeEntry) error
    // ...
}
```

`pkg/plugin` 和 `pkg/tape` 都依赖 `pkg/contract`，消除循环。

## 实现步骤

- [ ] 将 `BeforeToolCallArgs.Tool` 和 `.ToolCtx` 从 `any` 改为具体类型（已有 import）
- [ ] 将 `BatchBeforeToolCallArgs.Items` 从 `any` 改为 `[]tool.BatchCheckItem`
- [ ] 为 `ProvideApprover`、`ProvideChatInterface`、`SystemPrompt` 添加 typed wrapper
- [ ] 为 `RegisterCoreTools`、`RegisterExtendedTools` 添加 typed wrapper
- [ ] 为 `DiscoveredToolsCleared` 添加 typed wrapper
- [ ] 将 `InterceptInputArgs.TapeStore` / `TapeManager` 通过接口提取消除 `any`
- [ ] 修改 `opapolicy` 等消费者，移除 `_, _ :=` 断言，改为直接使用具体类型
- [ ] 在断言失败处添加显式错误日志（而非静默 return nil）

## 代价与风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| 阶段 1 破坏外部插件 | 极低 | 低 | 外部插件通过 RPC，不直接使用 typed_hooks |
| `pkg/contract` 包增加复杂度 | 中 | 低 | 仅放接口定义，不放实现 |
| Typed wrapper 与 raw HookFunc 共存期间混乱 | 中 | 中 | 标记 raw 注册方法为 deprecated，lint 检查 |

## 参考

- `pkg/plugin/hooks.go` — HookFunc 定义和 hook 常量
- `pkg/plugin/typed_hooks.go` — 现有 typed wrappers
- `plugins/opapolicy/opapolicy.go:98-103` — 断言失败被吞掉的示例
- `plugins/cli/cli.go:37-40` — 无类型 hook 注册示例
- [#11 Hook 系统无 panic 恢复](./11-hook-panic-and-race.md) — hook 稳定性的另一维度
- [#45 State Map 服务定位器](./45-state-service-locator.md) — 类型安全的另一维度
