# Agent Loop 深入分析

## 概述

Agent Loop 是 DMR 的核心执行循环，实现多轮 LLM 对话与自动工具执行。

## 执行流程

```
User Input
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. InterceptInput Hook (命令拦截)                           │
│    - 如果是命令（如 ,model.switch），直接处理并返回          │
│    - 否则继续                                                │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. 初始化                                                    │
│    - 收集工具 (核心 + 扩展)                                  │
│    - 创建 ToolContext                                        │
│    - 记录 session/start anchor (首次)                       │
│    - 追加 system prompt 和 user message 到 tape             │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Agent Loop (for step := 1; step <= maxSteps; step++)     │
│                                                              │
│    ┌─────────────────────────────────────────────────────┐  │
│    │ 3.1 预检查: Token 估计                              │  │
│    │     - 估计当前上下文 token 数                        │  │
│    │     - 如果超过阈值，触发预emptive compact           │  │
│    └─────────────────────────────────────────────────────┘  │
│    │                                                         │
│    ▼                                                         │
│    ┌─────────────────────────────────────────────────────┐  │
│    │ 3.2 LLM 调用 (ChatClient.RunTools)                  │  │
│    │     - 构建消息 (从 tape 读取历史)                    │  │
│    │     - 调用 LLM                                      │  │
│    │     - 如果错误是上下文溢出，处理 overflow            │  │
│    └─────────────────────────────────────────────────────┘  │
│    │                                                         │
│    ▼                                                         │
│    ┌─────────────────────────────────────────────────────┐  │
│    │ 3.3 处理响应                                        │  │
│    │                                                      │  │
│    │   Case 1: text response                             │  │
│    │     - 追加 assistant message 到 tape                │  │
│    │     - 返回结果，结束循环                            │  │
│    │                                                      │  │
│    │   Case 2: tool calls                                │  │
│    │     - 执行工具                                      │  │
│    │     - 追加 tool calls 和 results 到 tape            │  │
│    │     - 检查连续拒绝次数                              │  │
│    │     - 检查 token 使用，触发自动 handoff             │  │
│    │     - 继续下一轮                                    │  │
│    │                                                      │  │
│    │   Case 3: error                                     │  │
│    │     - 返回错误，结束循环                            │  │
│    └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
4. AfterAgentRun Hook (通知插件 Agent 运行完成)
```

## 核心代码

```go
func (a *Agent) Run(ctx context.Context, tapeName, prompt string, historyAfterEntryID int32) (*Result, error) {
    result, err := a.run(ctx, tapeName, prompt, historyAfterEntryID, nil, "")
    
    // 通知插件 Agent 运行完成
    if a.plugins != nil && a.plugins.Hooks().HasHook(plugin.HookAfterAgentRun) {
        _ = a.plugins.Hooks().CallAfterAgentRun(ctx, tapeName)
    }
    
    return result, err
}
```

## 详细循环逻辑

```go
func (a *Agent) run(ctx context.Context, tapeName, prompt string, historyAfterEntryID int32, mode *runMode, contextJSON string) (*Result, error) {
    // 1. 自动应用 per-tape 模型
    a.applyPerTapeModel(tapeName)
    
    // 2. 拦截命令
    if intercepted := a.interceptInput(ctx, prompt); intercepted != nil {
        return intercepted, nil
    }
    
    // 3. 收集工具
    tools := a.collectToolsWithDiscovery(ctx, tapeName)
    
    // 4. 创建 ToolContext
    toolCtx := tool.NewToolContext(ctx, tapeName, "")
    toolCtx.State["_runtime_workspace"] = a.config.Workspace
    toolCtx.State["_tape_store"] = a.tape.Store
    toolCtx.State["_tape_manager"] = a.tape
    toolCtx.State["_runtime_agent"] = a
    
    // 5. 解析插件上下文
    if contextJSON != "" {
        json.Unmarshal([]byte(contextJSON), &toolCtx.Context)
    }
    
    // 6. 初始化 tape
    a.initializeTape(tapeName, prompt)
    
    // 7. 主循环
    for step := 1; step <= maxSteps; step++ {
        // 7.1 预检查 compact
        if a.shouldPreemptiveCompact(tapeName, step) {
            a.CompactTapeWithName(ctx, tapeName, handoffName)
            continue
        }
        
        // 7.2 调用 LLM
        opts := client.ChatOpts{
            Prompt:       currentPrompt,
            SystemPrompt: systemPrompt,
            Tools:        tools,
            ToolContext:  toolCtx,
            Tape:         tapeName,
            Context:      tapeCtx,
        }
        
        result, err := chatClient.RunTools(ctx, opts)
        if err != nil {
            // 处理上下文溢出
            if isContextOverflowError(err) {
                if handled := a.handleContextOverflow(ctx, tapeName, step, currentPrompt); handled {
                    continue
                }
            }
            return nil, err
        }
        
        // 7.3 处理响应
        switch result.Kind {
        case "text":
            return a.handleTextResponse(tapeName, result, step)
            
        case "tools":
            shouldContinue := a.handleToolResponse(tapeName, result, step, &currentPrompt, &consecutiveDenies)
            if !shouldContinue {
                return &Result{...}, nil
            }
            
        case "error":
            return nil, fmt.Errorf("step %d: %s", step, result.Error.Message)
        }
    }
    
    return nil, fmt.Errorf("max steps reached (%d)", maxSteps)
}
```

## 工具执行处理

```go
func (a *Agent) handleToolResponse(tapeName string, result *core.ToolAutoResult, step int, currentPrompt *string, consecutiveDenies *int) bool {
    // 1. 构建工具结果消息
    var msgs []map[string]any
    
    // 添加 assistant message (带 tool_calls)
    assistantWithTools := map[string]any{
        "role":       "assistant",
        "content":    result.Text,
        "tool_calls": tcMaps,
    }
    msgs = append(msgs, assistantWithTools)
    
    // 添加 tool results
    for i, tr := range result.ToolResults {
        msgs = append(msgs, map[string]any{
            "role":         "tool",
            "tool_call_id": callID,
            "content":      truncateForProvider(fmt.Sprintf("%v", tr), maxChars),
        })
    }
    
    // 2. 记录到 tape
    a.tape.RecordChat(tape.RecordChatOpts{
        Tape:        tapeName,
        Messages:    msgs,
        ToolCalls:   result.ToolCalls,
        ToolResults: result.ToolResults,
    })
    
    // 3. 检查连续拒绝
    if allDenied(result.ToolResults) {
        (*consecutiveDenies)++
        if *consecutiveDenies >= 2 {
            // 全部拒绝，停止
            return false
        }
    } else {
        *consecutiveDenies = 0
    }
    
    // 4. 自动 handoff 检查
    if a.shouldAutoHandoff(tapeName, result.Usage) {
        handoffName := fmt.Sprintf("auto:token-threshold:%s", time.Now().UTC().Format("20060102-150405"))
        a.CompactTapeWithName(ctx, tapeName, handoffName)
        *currentPrompt = continueAfterCompactPrompt
        return true  // 继续循环
    }
    
    // 5. 设置下一轮提示
    *currentPrompt = "Continue based on the tool results above."
    return true
}
```

## 工具发现机制

```go
func (a *Agent) collectToolsWithDiscovery(ctx context.Context, tapeName string) []*tool.Tool {
    var tools []*tool.Tool
    
    // Step 1: 收集核心工具（始终加载）
    if a.plugins.Hooks().HasHook(plugin.HookRegisterCoreTools) {
        results, _ := a.plugins.Hooks().CallAll(ctx, plugin.HookRegisterCoreTools)
        tools = append(tools, plugin.CollectTools(results)...)
    }
    
    // Step 2: 添加 config.Tools（用户定义的工具）
    tools = append(tools, a.config.Tools...)
    
    // Step 3: 收集已发现的扩展工具
    if tapeName != "" {
        extendedTools := a.GetAllExtendedTools()
        for _, t := range extendedTools {
            if a.IsToolDiscovered(tapeName, t.Spec.Name) {
                tools = append(tools, t)
            }
        }
    }
    
    return tools
}
```

## 上下文溢出处理

```go
func (a *Agent) handleContextOverflow(ctx context.Context, tapeName string, step int, currentPrompt string) (bool, error) {
    if autoHandoffDone {
        return false, nil
    }
    
    // 尝试 compact
    handoffName := fmt.Sprintf("auto:overflow:%s", time.Now().UTC().Format("20060102-150405"))
    if err := a.CompactTapeWithName(ctx, tapeName, handoffName); err != nil {
        // Compact 失败，只创建 anchor
        a.Handoff(tapeName, handoffName, map[string]any{"reason": "context_overflow"})
    }
    
    autoHandoffDone = true
    return true, nil
}
```

## 自动 Handoff 策略

### Token 阈值检查

```go
func (a *Agent) shouldAutoHandoff(tapeName string, latestUsage map[string]any) bool {
    limit := a.handoffContextLimit(tapeName)
    if limit <= 0 || latestUsage == nil {
        return false
    }
    
    pt, ok := intFromUsageMap(latestUsage, "prompt_tokens")
    if !ok {
        return false
    }
    
    th := a.handoffThreshold(tapeName)
    return float64(pt) >= float64(limit) * th
}
```

### 配置

```toml
[agent]
max_token = 8000           # 上下文预算
handoff_threshold = 0.8    # 阈值比例
```

### 每个模型的配置

```toml
[[models]]
name = "gpt-4"
model = "gpt-4o"
max_token = 8000
handoff_threshold = 0.75
```

## 工具结果截断

为避免超过提供商输入限制，工具输出会被截断：

```go
func truncateForProvider(s string, maxChars int) string {
    if maxChars <= 0 || s == "" {
        return s
    }
    
    // 基于 rune 的截断（对 CJK 更安全）
    runes := []rune(s)
    if len(runes) <= maxChars {
        return s
    }
    
    // 保留头部和尾部
    half := maxChars / 2
    head := string(runes[:half])
    tail := string(runes[len(runes)-(maxChars-half):])
    
    truncationHint := fmt.Sprintf(
        "\n... [truncated %d chars] ...\n"+
        "⚠️ Tool output was truncated due to size limit.\n",
        len(runes)-maxChars,
    )
    
    return head + truncationHint + tail
}
```

## 压缩间隔控制

```go
func (a *Agent) shouldCompactNow(tapeName string, step int) bool {
    a.mu.Lock()
    defer a.mu.Unlock()
    
    lastCompact, hasCompacted := a.lastCompactStep[tapeName]
    
    // 如果当前 step < 上次记录的 step，是新对话周期
    if hasCompacted && step < lastCompact {
        delete(a.lastCompactStep, tapeName)
        return true
    }
    
    // 至少间隔 3 步
    return !hasCompacted || step-lastCompact >= 3
}
```
