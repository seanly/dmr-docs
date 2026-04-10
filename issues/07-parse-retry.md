# P3: LLM 输出解析失败重试（Parse Failure Auto-Retry）

> 灵感来源：TapeAgents `LLMOutputParsingFailureAction` + `SetNextNode(same_node)` 重试机制

## 问题

当 LLM 返回格式错误的 tool_call JSON 时（尤其是弱模型如 Ollama 本地模型），DMR 的 agent 循环将错误记录到 tape 并可能终止执行。没有自动修复重试机制。

TapeAgents 的 `StandardNode` 在解析失败时生成 `LLMOutputParsingFailureAction`，然后通过 `SetNextNode` 回到同一个 node 重试，并将解析错误作为上下文反馈给 LLM。

## 方案

在 `ChatClient.RunTools()` 中，如果 tool_call 的 JSON 参数解析失败：

```go
// pkg/client/chat.go RunTools 方法中

for _, call := range result.ToolCalls {
    var args map[string]any
    if err := json.Unmarshal([]byte(call.Arguments), &args); err != nil {
        if retryCount < maxParseRetries {
            retryCount++
            // 将错误信息作为 user message 发回
            continuationMsgs = append(continuationMsgs, map[string]any{
                "role":    "user",
                "content": fmt.Sprintf(
                    "Your tool call '%s' had invalid JSON arguments: %s\nOriginal: %s\nPlease retry with valid JSON.",
                    call.Function.Name, err.Error(), call.Arguments,
                ),
            })
            continue  // 重新循环，发新的 LLM 请求
        }
        // 超过重试次数，记录错误
        return Result{Kind: "error", Error: err}
    }
}
```

配置：
```toml
[agent]
max_parse_retries = 2  # 默认重试 2 次
```

## 解决的问题

- 弱模型（Ollama 本地）的 JSON 格式不稳定导致任务中断
- 某些模型偶发的 tool_call 格式异常（参数未闭合等）
- 提升 agent 的鲁棒性，减少人工干预

## 代价

- 约 20 行改动，局部在 `RunTools` 内
- 重试增加 LLM 调用次数（但只在解析失败时触发，正常流程无开销）
- 需要限制重试次数防止死循环

## 参考

- TapeAgents `LLMOutputParsingFailureAction`：`tapeagents/core.py:163-175`
- TapeAgents `StandardNode.parse_completion` 重试：`tapeagents/nodes.py:260-290`
- DMR tool_call 解析：`pkg/client/chat.go` RunTools 方法
