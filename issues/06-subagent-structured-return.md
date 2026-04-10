# P3: Subagent 结构化返回（Subagent Structured Return）

> 灵感来源：TapeAgents `TapeViewStack` 跨 agent 状态可见性

## 问题

DMR subagent 执行后，parent 只收到一个 opaque 文本字符串（`res.Output`）。Parent 无法知道：
- Child 执行了哪些工具
- 花了多少步 / 多少 token
- 是否有错误或被 OPA 拒绝的操作
- 执行耗时

这限制了 parent agent 的决策能力（"上个 subagent 失败了三次，我应该换策略"）。

## 方案

subagent 返回时，除了文本 output，附带结构化摘要：

```go
type SubagentResult struct {
    Output     string       `json:"output"`
    Summary    ExecSummary  `json:"_summary"`
}

type ExecSummary struct {
    Steps        int      `json:"steps"`
    ToolCalls    []string `json:"tool_calls"`    // 调用了哪些工具
    PromptTokens int     `json:"prompt_tokens"`
    CompletionTokens int `json:"completion_tokens"`
    DurationMs   int64   `json:"duration_ms"`
    HasError     bool    `json:"has_error"`
    DeniedCalls  int     `json:"denied_calls"`  // OPA 拒绝的次数
}
```

返回给 parent 的 tool_result 格式：
```json
{
    "output": "文件读取完成，内容为...",
    "_summary": {
        "steps": 5,
        "tool_calls": ["fsRead", "shell"],
        "prompt_tokens": 3200,
        "duration_ms": 8500,
        "has_error": false
    }
}
```

Parent agent 可以利用 summary 做更好的决策，也可以忽略它（LLM 通常会自动关注 output 字段）。

## 解决的问题

- Parent agent 根据 child 执行情况调整策略
- 运维人员理解 subagent 链的资源消耗
- 配合 P1（Tape 谱系）提供完整的执行洞察

## 参考

- TapeAgents `TapeView.outputs_by_subagent`：`tapeagents/view.py:55`
- DMR `RunSubagent` 返回：`pkg/agent/subagent.go`
