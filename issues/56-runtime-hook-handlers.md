# P2: 运行时 Hook 处理器 — HTTP Webhook 与 Prompt Hook

> DMR 当前所有 hook 处理器都是编译时绑定的 Go 插件，新增或修改策略逻辑需要重新编译。Claude Code 支持 command / http / prompt / agent 四种运行时 hook 类型。DMR 应支持 HTTP webhook 和 Prompt hook，让用户无需重编译即可扩展安全策略。

## 问题

### 1. 编译时绑定限制扩展性

当前 DMR 的插件系统（`pkg/plugin/`）要求所有 hook 处理器实现 Go 接口并在 `cmd/dmr/plugins.go` 中注册。这意味着：

- 添加一个简单的审计 webhook 需要写 Go 代码、编译、部署
- 企业安全团队无法用自己熟悉的语言（Python、Node.js）编写策略
- 不同环境（dev / staging / prod）无法通过配置切换不同的策略后端

### 2. 缺少轻量级策略扩展机制

OPA Rego 适合确定性规则，但某些场景需要：

- **调外部 API**：查询 CMDB 确认目标服务器是否在维护窗口内
- **模糊判断**：用 LLM 分析命令是否有数据外泄风险（OPA 写不了这种语义规则）
- **审计上报**：将每次工具调用记录到企业 SIEM 系统

这些都需要运行时可配置的处理器。

## 设计方案

### 1. Hook 处理器类型扩展

```toml
# ~/.dmr/config.toml

# 现有：Go 插件（编译时）
[[plugins]]
name = "opa_policy"
enabled = true

# 新增：运行时 hook 处理器
[[hooks.before_tool_call]]
name = "audit-webhook"
type = "http"
url = "https://siem.corp.example/api/audit"
timeout = "5s"
async = true  # 异步，不阻塞工具执行
headers = { "Authorization" = "Bearer ${DMR_AUDIT_TOKEN}" }

[[hooks.before_tool_call]]
name = "security-check"
type = "http"
url = "https://security-gateway.corp.example/api/check"
timeout = "10s"
async = false  # 同步，等待响应决策

[[hooks.before_tool_call]]
name = "exfil-detector"
type = "prompt"
model = "haiku"  # 使用轻量模型
prompt = |
  Analyze the following tool call and determine if it could be
  exfiltrating sensitive data. Respond with JSON:
  {"safe": true/false, "reason": "..."}

  Tool: {{.Tool}}
  Arguments: {{.Args}}
timeout = "15s"
```

### 2. HTTP Webhook 处理器

```go
// pkg/plugin/runtime_hooks/http_hook.go

type HTTPHook struct {
    Name    string
    URL     string
    Timeout time.Duration
    Async   bool
    Headers map[string]string
}

// Request payload
type HTTPHookRequest struct {
    Event   string         `json:"event"`    // "before_tool_call"
    Tool    string         `json:"tool"`
    Args    map[string]any `json:"args"`
    Context struct {
        Tape      string `json:"tape"`
        Workspace string `json:"workspace"`
    } `json:"context"`
}

// Response payload
type HTTPHookResponse struct {
    Action  string         `json:"action"`   // "allow" / "deny" / "" (defer)
    Reason  string         `json:"reason"`
    Context string         `json:"context"`  // additional context to inject
}

func (h *HTTPHook) Execute(ctx context.Context, req HTTPHookRequest) (*HTTPHookResponse, error) {
    ctx, cancel := context.WithTimeout(ctx, h.Timeout)
    defer cancel()

    body, _ := json.Marshal(req)
    httpReq, _ := http.NewRequestWithContext(ctx, "POST", h.URL, bytes.NewReader(body))
    for k, v := range h.Headers {
        httpReq.Header.Set(k, os.ExpandEnv(v))
    }

    resp, err := http.DefaultClient.Do(httpReq)
    if err != nil {
        if h.Async {
            slog.Warn("async hook failed", "name", h.Name, "error", err)
            return nil, nil // async hook failure doesn't block
        }
        return nil, fmt.Errorf("hook %s: %w", h.Name, err)
    }

    var result HTTPHookResponse
    json.NewDecoder(resp.Body).Decode(&result)
    return &result, nil
}
```

### 3. Prompt Hook 处理器

```go
// pkg/plugin/runtime_hooks/prompt_hook.go

type PromptHook struct {
    Name     string
    Model    string        // LLM model to use
    Template string        // prompt template with {{.Tool}}, {{.Args}}
    Timeout  time.Duration
}

type PromptHookResult struct {
    Safe   bool   `json:"safe"`
    Reason string `json:"reason"`
}

func (h *PromptHook) Execute(ctx context.Context, tool string, args map[string]any) (*BeforeToolCallResult, error) {
    // 1. 渲染 prompt 模板
    prompt := h.renderTemplate(tool, args)

    // 2. 调用 LLM（使用配置的 model）
    response, err := h.llm.Complete(ctx, prompt)
    if err != nil {
        return nil, fmt.Errorf("prompt hook %s: %w", h.Name, err)
    }

    // 3. 解析 JSON 响应
    var result PromptHookResult
    if err := json.Unmarshal([]byte(response), &result); err != nil {
        slog.Warn("prompt hook returned non-JSON", "name", h.Name, "response", response)
        return nil, nil // 解析失败不阻塞
    }

    if !result.Safe {
        return nil, fmt.Errorf("denied by %s: %s", h.Name, result.Reason)
    }
    return nil, nil
}
```

### 4. 运行时 Hook 注册

```go
// pkg/plugin/runtime_hooks/registry.go

func LoadRuntimeHooks(cfg *config.Config, hookRegistry *plugin.HookRegistry) error {
    for _, hc := range cfg.Hooks.BeforeToolCall {
        switch hc.Type {
        case "http":
            hook := &HTTPHook{Name: hc.Name, URL: hc.URL, ...}
            hookRegistry.RegisterBeforeToolCall(hc.Name, 50, hook.AsBeforeToolCallFunc())
        case "prompt":
            hook := &PromptHook{Name: hc.Name, Model: hc.Model, ...}
            hookRegistry.RegisterBeforeToolCall(hc.Name, 50, hook.AsBeforeToolCallFunc())
        }
    }
    return nil
}
```

运行时 hook 的默认 priority 为 50，低于 OPA 插件的 100，确保 OPA deny 规则始终优先。

### 5. 安全约束

- HTTP hook 只允许 HTTPS URL（与 OPA 策略加载一致）
- Prompt hook 的 LLM 调用有独立的 token 预算和超时
- Async hook 的失败不阻塞工具执行，但记录审计日志
- 运行时 hook 受 managed 配置的 `allow_managed_hooks_only` 约束

## 实现步骤

- [ ] 定义运行时 hook 配置结构（`pkg/config/hooks.go`）
- [ ] 实现 HTTP webhook 处理器（`pkg/plugin/runtime_hooks/http_hook.go`）
- [ ] 实现 Prompt hook 处理器（`pkg/plugin/runtime_hooks/prompt_hook.go`）
- [ ] 实现运行时 hook 注册逻辑（`pkg/plugin/runtime_hooks/registry.go`）
- [ ] 在 `cmd/dmr/agent_builder.go` 中初始化运行时 hook
- [ ] 环境变量展开支持（`${DMR_AUDIT_TOKEN}` 等）
- [ ] 配置热重载支持（与 OPA 策略热重载一致）
- [ ] 添加 `dmr hooks list` 命令显示所有已注册 hook（编译时 + 运行时）
- [ ] 测试：HTTP hook 超时、async 失败、prompt hook 解析错误
- [ ] 文档：运行时 hook 配置指南 + 示例

## 代价与风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| HTTP hook 目标服务不可用导致工具调用卡住 | 中 | 高 | 严格的超时控制；async hook 不阻塞；circuit breaker 模式 |
| Prompt hook 产生不确定结果（LLM 幻觉） | 中 | 中 | Prompt hook 只能 deny（保守侧），不能 allow 绕过 OPA deny |
| HTTP hook 暴露敏感信息（工具参数含密码） | 中 | 高 | 配置中支持 `redact_fields` 脱敏；HTTPS only |
| 运行时 hook 性能开销（每次工具调用都触发） | 低 | 中 | async 模式、缓存、条件触发（matcher 过滤工具类型） |
| 增加配置复杂度 | 确定 | 低 | 运行时 hook 完全可选；不配置时行为不变 |

## 参考

- Claude Code hook 类型：`command` / `http` / `prompt` / `agent`
- Claude Code prompt hook — 用 LLM 做策略判断
- `plugins/opapolicy/engine_build.go` — HTTPS URL 策略加载（类似的安全约束）
- `pkg/plugin/hooks.go` — HookRegistry 现有注册机制
- `cmd/dmr/plugins.go` — 现有编译时插件注册
