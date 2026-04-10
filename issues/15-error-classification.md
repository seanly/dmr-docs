# P2: 错误分类过宽导致错误的 Compact 触发

> 灵感来源：TapeAgents `LLMOutputParsingFailureAction` 的精确错误分类

## 问题

`pkg/agent/reactive_handoff.go:161-163`：

```go
func isContextOverflowError(err error) bool {
    return err != nil && core.IsErrorKind(err, core.ErrInvalidInput)
}
```

所有 `ErrInvalidInput` 都被当作"上下文溢出"，触发 compact + retry。

但 `ErrInvalidInput` 的分类范围过广（`pkg/core/execution.go:254`）：

```go
case strings.Contains(msg, "invalid") || strings.Contains(msg, "validation"):
    return ErrInvalidInput
```

这意味着以下错误都会被误判为"上下文溢出"：
- `"invalid JSON in tool arguments"` — 工具参数格式错误
- `"validation error: temperature must be between 0 and 2"` — 参数校验失败
- `"invalid model name"` — 模型名错误

**后果**：这些非溢出错误触发了不必要的 compact，compact 后重试**同一个 LLM 请求**（参数仍然错误），再次失败，再次 compact... 直到 `maxSteps` 耗尽。浪费步骤、丢失上下文、用户困惑。

## TapeAgents 的做法

TapeAgents 对 LLM 输出错误有精确分类：

```python
# 解析失败 → 专门的 step 类型
LLMOutputParsingFailureAction(error=str(e), llm_output=raw_output)

# 费率限制 → 重试
except litellm.RateLimitError: retry with backoff

# 超时 → 重试
except litellm.Timeout: retry with backoff

# 其他 → 直接抛出，不假装能处理
```

关键区别：**不把多种错误混为一谈**。

## 修复方案

### 1. 新增专门的错误类型

```go
// pkg/core/errors.go

const (
    ErrContextOverflow ErrorKind = "context_overflow"  // 新增
    ErrInvalidInput    ErrorKind = "invalid_input"      // 保留
)
```

### 2. 精确分类上下文溢出

```go
func classifyError(code int, msg string) ErrorKind {
    // 先检查明确的溢出信号
    overflowPatterns := []string{
        "maximum context length",
        "context_length_exceeded",
        "max_tokens",
        "too many tokens",
        "context window",
        "prompt is too long",
    }
    for _, p := range overflowPatterns {
        if strings.Contains(strings.ToLower(msg), p) {
            return ErrContextOverflow
        }
    }
    
    // 其他 invalid/validation 错误
    if strings.Contains(msg, "invalid") || strings.Contains(msg, "validation") {
        return ErrInvalidInput
    }
    // ...
}
```

### 3. 修改 reactive handoff 判断

```go
func isContextOverflowError(err error) bool {
    return err != nil && core.IsErrorKind(err, core.ErrContextOverflow)
}
```

## 参考

- TapeAgents 精确错误分类：`tapeagents/llms/lite.py:58-97`
- TapeAgents 解析失败处理：`tapeagents/nodes.py:389-454`
- DMR 错误分类：`pkg/core/execution.go:230-260`
- DMR reactive handoff：`pkg/agent/reactive_handoff.go`
