# P2: HTTP 5xx 全部不重试 + 流式中断不可恢复

> 灵感来源：TapeAgents `LiteLLM._generate()` 的重试策略

## 问题 1：HTTP 502/503 被归类为不可重试

`pkg/core/execution.go:239`：

```go
case code >= 500:
    return ErrProvider  // 不可重试，直接 fallback 到下一个模型
```

HTTP 502 Bad Gateway 和 503 Service Unavailable 是典型的**瞬态错误**（网关抖动、服务重启）。应该先重试，失败后才 fallback。

当前行为：一次 502 → 立即切换模型 → 可能切到更贵/更慢的 fallback 模型。

## 问题 2：流式中断不可恢复

`pkg/core/execution.go` 的 `RunChatStream`：

```go
func (c *LLMCore) RunChatStream(ctx context.Context, opts RunChatOpts) (<-chan StreamChunk, error) {
    // 只在连接建立阶段重试
    // 一旦流开始，中间断开 → chunk.Err 传给调用者 → 无法重试
}
```

流式响应中途断开（网络抖动、provider 超时）后，`chat.go` 的 `Stream()` 和 `StreamEvents()` 收到 error chunk，但**无法触发 retry/fallback**。已接收的 partial response 丢失。

## TapeAgents 的做法

```python
# tapeagents/llms/lite.py
while True:
    try:
        response = litellm.completion(...)
        break
    except litellm.RateLimitError:
        if retry_count >= self.max_retries:
            raise
        delay = self.base_delay * (2 ** (retry_count - 1))
        time.sleep(delay)
        retry_count += 1
    except litellm.Timeout:
        # 同样的重试逻辑
```

TapeAgents 也没有处理流式中断，但至少连接级错误有完整的重试。

## 修复方案

### 1. 细化 5xx 分类

```go
func classifyByCode(code int) ErrorKind {
    switch {
    case code == 429:
        return ErrTemporary  // rate limit
    case code == 502, code == 503, code == 504:
        return ErrTemporary  // 瞬态网关错误，应重试
    case code >= 500:
        return ErrProvider   // 其他 5xx，真正的 provider 错误
    // ...
    }
}
```

### 2. 流式断线重试（降级为非流式）

```go
func (c *ChatClient) StreamWithFallback(ctx context.Context, opts RunChatOpts) (<-chan string, error) {
    ch, state, err := c.Stream(ctx, opts)
    if err != nil {
        return nil, err
    }
    
    // 监控流完成状态
    outCh := make(chan string)
    go func() {
        defer close(outCh)
        for text := range ch {
            outCh <- text
        }
        
        // 如果流中断且有瞬态错误
        if state.Err != nil && isTransientStreamError(state.Err) {
            // 降级为非流式重试
            result, retryErr := c.core.RunChat(ctx, opts)
            if retryErr == nil {
                outCh <- result.Text
            }
        }
    }()
    return outCh, nil
}
```

## 参考

- TapeAgents 重试逻辑：`tapeagents/llms/lite.py:58-97`
- DMR 错误分类：`pkg/core/execution.go:230-260`
- DMR 流式处理：`pkg/client/chat.go:150-201`
