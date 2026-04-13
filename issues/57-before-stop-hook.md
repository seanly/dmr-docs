# P2: BeforeStop Hook — Agent 停止前检查

> Claude Code 的 `Stop` hook 可以阻止 agent 结束响应，强制其继续工作（exit code 2 阻止停止）。DMR 缺少类似机制，无法在 agent 完成对话时进行检查。在自动化场景中（如 CI/CD、定时任务），需要确保 agent 完成了所有必要步骤才允许停止。

## 问题

### 1. Agent 可能过早停止

LLM 可能在以下情况提前结束对话：

- 修改了代码但没有运行测试
- 创建了文件但没有验证格式
- 执行了部分步骤后认为"任务完成"
- 遇到错误后放弃而不是重试

在交互式场景中，用户可以手动要求 agent 继续。但在非交互场景（`dmr run -p "..."` / brain / cron 任务）中，agent 停止就是最终结果。

### 2. 无法实施"完成条件"

某些场景需要 agent 满足特定条件后才能停止：

| 场景 | 完成条件 |
|------|---------|
| CI 代码审查 | 必须对每个修改的文件给出评审意见 |
| 自动修复 | 修改代码后必须运行 `go test` 且通过 |
| 定时报告 | 必须生成并保存报告文件 |
| 多步骤工作流 | 必须按顺序完成所有步骤 |

当前没有机制在 agent 停止前检查这些条件。

## 设计方案

### 1. 新增 BeforeStop Hook 事件

```go
// pkg/plugin/typed_hooks.go

// BeforeStopArgs is passed to BeforeStop hook handlers.
type BeforeStopArgs struct {
    // Tape is the conversation tape name.
    Tape string

    // Reason is why the agent is stopping ("end_turn" / "max_steps" / "error").
    Reason string

    // StepsCompleted is the number of agent steps executed.
    StepsCompleted int

    // ToolsUsed is the list of tools invoked during this turn.
    ToolsUsed []string

    // HookActive is true if this stop was already intercepted by a BeforeStop hook.
    // Use this to prevent infinite loops.
    HookActive bool
}

// BeforeStopResult is the return value from BeforeStop hooks.
type BeforeStopResult struct {
    // Block prevents the agent from stopping. The agent will receive
    // ContinuePrompt as additional context and continue working.
    Block bool

    // ContinuePrompt is injected when Block is true.
    // Example: "You haven't run tests yet. Please run `go test ./...` before finishing."
    ContinuePrompt string
}

// BeforeStopFunc is the typed signature.
type BeforeStopFunc func(ctx context.Context, args BeforeStopArgs) (*BeforeStopResult, error)
```

### 2. Agent 循环集成

```go
// pkg/agent/agent.go — agent 主循环

func (a *Agent) Run(ctx context.Context) error {
    for step := 0; step < a.maxSteps; step++ {
        response, err := a.llm.Complete(ctx, messages)
        if err != nil {
            return err
        }

        // 处理工具调用...

        // 检查是否要停止
        if response.StopReason == "end_turn" {
            result, err := a.hooks.CallBeforeStop(ctx, plugin.BeforeStopArgs{
                Tape:           a.tape.Name(),
                Reason:         "end_turn",
                StepsCompleted: step + 1,
                ToolsUsed:      a.toolsUsedThisTurn,
                HookActive:     a.stopHookActive,
            })
            if err != nil {
                return err
            }
            if result != nil && result.Block && !a.stopHookActive {
                // Hook 阻止停止，注入提示继续
                a.stopHookActive = true
                messages = append(messages, systemMessage(result.ContinuePrompt))
                continue // 继续 agent 循环
            }
            break // 允许停止
        }
    }
    return nil
}
```

### 3. 示例：测试检查 Hook

```go
// plugins/stop_guard/plugin.go

func (p *StopGuardPlugin) BeforeStop(ctx context.Context, args plugin.BeforeStopArgs) (*plugin.BeforeStopResult, error) {
    // 防止无限循环
    if args.HookActive {
        return nil, nil
    }

    // 检查是否使用了写操作但没有运行测试
    hasWrite := slices.ContainsFunc(args.ToolsUsed, func(t string) bool {
        return t == "fsWrite" || t == "fsReplace"
    })
    hasTest := slices.ContainsFunc(args.ToolsUsed, func(t string) bool {
        return t == "shell" // 需要更细粒度的检查
    })

    if hasWrite && !hasTest {
        return &plugin.BeforeStopResult{
            Block:          true,
            ContinuePrompt: "You modified files but did not run tests. Please run the project's test suite before finishing.",
        }, nil
    }

    return nil, nil
}
```

### 4. 配置化的完成条件

```toml
# ~/.dmr/config.toml

[plugins.stop_guard]
enabled = true

[plugins.stop_guard.config]
# 如果使用了写操作，停止前必须运行的命令
require_after_write = ["go test ./..."]

# 最大阻止次数（防止无限循环）
max_blocks = 3

# 仅在非交互模式下生效
only_non_interactive = true
```

## 实现步骤

- [ ] 定义 `BeforeStopArgs`、`BeforeStopResult`、`BeforeStopFunc`（`pkg/plugin/typed_hooks.go`）
- [ ] 在 `HookRegistry` 中注册 `BeforeStop` 事件
- [ ] 在 agent 主循环的停止逻辑中调用 `CallBeforeStop`
- [ ] 实现 `HookActive` 防无限循环机制
- [ ] 实现 `stop_guard` 示例插件
- [ ] 设置最大阻止次数上限（硬编码 + 可配置）
- [ ] 测试：正常停止、阻止后继续、无限循环防护、max_steps 停止
- [ ] 文档：BeforeStop hook 使用指南

## 代价与风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| 无限循环：hook 反复阻止停止 | 中 | 高 | `HookActive` 标志 + `max_blocks` 上限（默认 3 次）|
| 增加 agent 运行成本（额外步骤消耗 token） | 确定 | 低 | `only_non_interactive` 选项；用户可关闭 |
| BeforeStop + max_steps 交互：到达 max_steps 时不应阻止 | 低 | 中 | `Reason == "max_steps"` 时跳过 hook |
| Hook 执行错误导致 agent 无法停止 | 低 | 高 | hook 执行错误视为"允许停止"（fail-open） |

## 参考

- Claude Code `Stop` hook — exit code 2 阻止停止；`stop_hook_active` 防循环
- `pkg/agent/` — agent 主循环
- `pkg/plugin/typed_hooks.go` — 现有 hook 类型定义
