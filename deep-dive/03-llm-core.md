# LLM Core 执行引擎深入分析

## 概述

LLM Core 是 DMR 的执行引擎，负责与 LLM 提供商通信，实现重试和降级逻辑。

## 架构

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   ChatClient    │────▶│    LLMCore      │────▶│ OpenAI Client   │
│  (pkg/client)   │     │  (pkg/core)     │     │ (pkg/openai)    │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                               │
                               ▼
                        ┌─────────────────┐
                        │  Retry/Fallback │
                        │    Engine       │
                        └─────────────────┘
```

## LLMCore

```go
type LLMCore struct {
    model           string           // 主模型
    fallbacks       []string         // 降级模型列表
    maxRetries      int              // 最大重试次数
    apiKey          string           // API 密钥
    apiBase         string           // API 基础 URL
    verbose         int              // 详细级别
    errorClassifier func(error) ErrorKind  // 错误分类器
    
    mu          sync.Mutex
    clientCache map[string]ChatProvider  // 按模型缓存客户端
    
    // OAuth2 客户端凭证
    tokenURL     string
    clientID     string
    clientSecret string
    headers      map[string]string  // 额外 HTTP 头
}
```

## ChatProvider 接口

```go
type ChatProvider interface {
    ChatCompletion(ctx context.Context, req ChatRequest) (*ChatResponse, error)
    ChatCompletionStream(ctx context.Context, req ChatRequest) (<-chan StreamChunk, error)
}
```

## 重试与降级策略

```
RunChat:
    models = [primary] + fallbacks
    
    for each model in models:
        client = GetClient(model)
        
        for attempt in 0..maxRetries:
            resp, err = client.ChatCompletion(req)
            
            if err == nil:
                return resp, nil
            
            kind = ClassifyError(err)
            
            if kind != ErrTemporary:
                break  // 非临时错误，尝试下一个模型
                
    return "all models exhausted"
```

## 错误分类

### 错误类型 (ErrorKind)

```go
const (
    ErrInvalidInput ErrorKind = "invalid_input"  // 输入错误
    ErrConfig       ErrorKind = "config"          // 配置错误
    ErrProvider     ErrorKind = "provider"        // 提供商错误
    ErrTool         ErrorKind = "tool"            // 工具错误
    ErrTemporary    ErrorKind = "temporary"       // 临时错误（可重试）
    ErrNotFound     ErrorKind = "not_found"       // 未找到
    ErrDenied       ErrorKind = "denied"          // 被拒绝
    ErrUnknown      ErrorKind = "unknown"         // 未知错误
)
```

### 四层错误分类策略

```go
func (c *LLMCore) defaultClassifyError(err error) ErrorKind {
    // 第1层: 检查 HTTP 状态码
    if code := extractHTTPStatus(err); code > 0 {
        return classifyByHTTPStatus(code)
    }
    
    // 第2层: 文本匹配
    return classifyByTextSignature(err)
}
```

#### HTTP 状态码映射

| 状态码 | ErrorKind |
|--------|-----------|
| 401/403 | ErrProvider |
| 429 | ErrTemporary |
| 400 | ErrInvalidInput |
| 404 | ErrNotFound |
| 5xx | ErrProvider |

#### 文本签名匹配

```go
func classifyByTextSignature(err error) ErrorKind {
    msg := strings.ToLower(err.Error())
    switch {
    case contains(msg, "rate limit", "too many requests"):
        return ErrTemporary
    case contains(msg, "timeout", "timed out"):
        return ErrTemporary
    case contains(msg, "connection refused", "connection reset"):
        return ErrTemporary
    case contains(msg, "invalid", "validation", "exceeded limit"):
        return ErrInvalidInput
    case contains(msg, "not found"):
        return ErrNotFound
    case contains(msg, "unauthorized", "forbidden"):
        return ErrProvider
    default:
        return ErrUnknown
    }
}
```

## 客户端工厂

```go
func (c *LLMCore) GetClient(model string) ChatProvider {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    if client, ok := c.clientCache[model]; ok {
        return client
    }
    
    // 创建新客户端
    oc := openai.ClientConfig{
        APIKey:  c.apiKey,
        BaseURL: c.apiBase,
        Headers: c.headers,
    }
    
    // OAuth2 客户端凭证支持
    if c.tokenURL != "" && c.clientID != "" && c.clientSecret != "" {
        oc.APIKey = ""
        oc.BearerToken = auth.ClientCredentialsTokenSource(...)
    }
    
    client := openai.NewClient(oc)
    c.clientCache[model] = client
    return client
}
```

## 请求构建

```go
func (c *LLMCore) buildChatRequest(model string, opts RunChatOpts) ChatRequest {
    messages := make([]Message, 0, len(opts.Messages))
    
    for _, m := range opts.Messages {
        role := getString(m, "role")
        content := getString(m, "content")
        
        // 处理空内容的 assistant 消息
        if role == "assistant" && content == "" {
            if _, hasTC := m["tool_calls"]; !hasTC {
                continue
            }
        }
        
        msg := Message{
            Role:    role,
            Content: content,
        }
        
        // 处理 tool_call_id
        if toolCallID, ok := m["tool_call_id"].(string); ok {
            msg.ToolCallID = toolCallID
        }
        
        // 规范化 tool_calls 参数
        if rawToolCalls, ok := m["tool_calls"]; ok {
            // 规范化 function.arguments 为有效 JSON
            msg.ToolCalls = normalizeToolCalls(rawToolCalls)
        }
        
        messages = append(messages, msg)
    }
    
    return ChatRequest{
        Model:       model,
        Messages:    messages,
        Tools:       opts.Tools,
        ToolChoice:  opts.ToolChoice,
        MaxTokens:   opts.MaxTokens,
        Temperature: opts.Temperature,
        TopP:        opts.TopP,
    }
}
```

## 参数规范化

DMR 存储工具调用参数为字符串，不同提供商可能产生非严格 JSON。执行前需要规范化：

```go
func normalizeFunctionArguments(v any) string {
    switch x := v.(type) {
    case nil:
        return "{}"
    case string:
        return normalizeArgumentsString(x)
    default:
        // 如果参数已经是结构化值，规范化为 JSON 对象字符串
        if m, ok := x.(map[string]any); ok {
            return string(mustMarshal(m))
        }
        return string(mustMarshal(map[string]any{"raw": x}))
    }
}

func normalizeArgumentsString(s string) string {
    s = strings.TrimSpace(s)
    if s == "" {
        return "{}"
    }
    
    // 尝试解析 JSON
    var parsed any
    if err := json.Unmarshal([]byte(s), &parsed); err == nil {
        // 已经是 JSON 对象
        if m, ok := parsed.(map[string]any); ok {
            return string(mustMarshal(m))
        }
        
        // 特殊情况: s 是包含 JSON 的 JSON 字符串
        if inner, ok := parsed.(string); ok && inner != "" {
            var innerParsed any
            if err2 := json.Unmarshal([]byte(inner), &innerParsed); err2 == nil {
                if m, ok := innerParsed.(map[string]any); ok {
                    return string(mustMarshal(m))
                }
            }
        }
        
        return string(mustMarshal(map[string]any{"raw": parsed}))
    }
    
    // 不是有效 JSON，包装为原始字符串
    return string(mustMarshal(map[string]any{"raw": s}))
}
```

## 流式响应

```go
func (c *LLMCore) RunChatStream(ctx context.Context, opts RunChatOpts) (<-chan StreamChunk, error) {
    models := append([]string{c.model}, c.fallbacks...)
    
    for _, model := range models {
        client := c.GetClient(model)
        req := c.buildChatRequest(model, opts)
        req.Stream = true
        req.StreamOptions = &StreamOptions{IncludeUsage: true}
        
        for attempt := range c.maxRetries + 1 {
            ch, err := client.ChatCompletionStream(ctx, req)
            if err == nil {
                return ch, nil
            }
            // ... 重试逻辑
        }
    }
    
    return nil, NewError(ErrUnknown, "all models exhausted", nil)
}
```

## 详细级别 (Verbose)

| 级别 | 行为 |
|------|------|
| 0 | 安静模式 |
| 1 | 记录请求/响应摘要 |
| 2 | 记录调试信息 |
| 3 | 记录完整请求/响应体 |

```go
if c.verbose >= 1 {
    slog.Info("LLM request", "model", model, "messages", len(req.Messages), "tools", len(req.Tools))
}
if c.verbose >= 3 {
    if data, err := json.MarshalIndent(req, "", "  "); err == nil {
        slog.Debug("request body", "body", string(data))
    }
}
```
