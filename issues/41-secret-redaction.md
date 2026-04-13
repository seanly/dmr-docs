# P0: Secret Redaction — 工具输出中的凭据脱敏

## 问题

DMR 的 `credential_bindings` 机制（`plugins/credentials/`）可以将加密存储的凭据注入到 shell 命令的环境变量中。但注入后，工具的**输出**可能包含凭据明文值，这些值会被写入 tape 并发送给 LLM。

攻击路径：

```
1. 用户配置 credential_bindings: {"API_TOKEN": "my-prod-token"}
2. LLM 调用 shell: curl -H "Authorization: Bearer $API_TOKEN" https://api.example.com
3. 工具输出: {"token": "sk-prod-xxxx", "response": ...}  ← 凭据出现在 HTTP 响应中
4. 输出被写入 tape（持久化到 SQLite）
5. 输出被发送给 LLM（可能被用于后续工具调用，泄露到其他 API）
```

更严重的场景：

- `env` 或 `printenv` 命令直接列出所有环境变量（包含注入的凭据）
- 调试模式的工具输出可能包含完整的 HTTP 请求头
- 错误日志中可能包含凭据值

### 现状

`TODO.md` 已将此标记为高优先级。`docs/issues/secret-redaction-via-credential-match.md` 在 dmr 主仓库中有基础设计，但尚未实现。

## 设计方案

### 核心原则

**在工具输出进入 tape 和 LLM 之前，将已知凭据值替换为占位符。**

### 1. Redaction Engine

```go
// pkg/credentials/redactor.go

type Redactor struct {
    mu       sync.RWMutex
    patterns map[string]string  // secret value -> credential name
}

// Register 注册需要脱敏的凭据值
func (r *Redactor) Register(credName, secretValue string) {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.patterns[secretValue] = credName
}

// Redact 替换输出中的凭据值
func (r *Redactor) Redact(output string) string {
    r.mu.RLock()
    defer r.mu.RUnlock()
    
    for secret, name := range r.patterns {
        if len(secret) < 4 {
            continue // 太短的值容易误匹配
        }
        placeholder := fmt.Sprintf("[REDACTED:%s]", name)
        output = strings.ReplaceAll(output, secret, placeholder)
    }
    return output
}
```

### 2. 集成点：工具执行后

```go
// pkg/tool/executor.go — executeOne 方法中

func (e *Executor) executeOne(ctx context.Context, tc *ToolCall, t *Tool) *ToolResult {
    result, err := t.Handler(tc.Context, tc.Args)
    
    // 脱敏：在结果写入 tape 之前
    if resultStr, ok := result.(string); ok {
        result = e.redactor.Redact(resultStr)
    }
    
    return &ToolResult{...}
}
```

### 3. 集成点：credential_bindings 注入时注册

```go
// plugins/credentials/hooks.go — BeforeToolCall hook 中

func (p *CredentialsPlugin) beforeToolCall(ctx context.Context, args plugin.BeforeToolCallArgs) error {
    for envName, credName := range bindings {
        secret, err := p.store.Get(credName)
        if err != nil {
            continue
        }
        os.Setenv(envName, secret)
        
        // 注册到全局 redactor
        p.redactor.Register(credName, secret)
    }
    return nil
}
```

### 4. 额外防护层

**环境变量命令拦截**：在 OPA 策略中默认拒绝 `env`、`printenv`、`set` 等直接列出环境变量的命令。

```rego
# policies/default.rego
deny_tool_call {
    input.tool == "shell"
    env_dump_commands := {"env", "printenv", "set", "export"}
    cmd := trim_space(input.args.command)
    some pattern in env_dump_commands
    startswith(cmd, pattern)
}
```

**Tape 写入脱敏**：即使 Redactor 漏掉，tape 写入时做第二层检查：

```go
// pkg/tape/manager.go — Append 方法中
func (m *Manager) Append(ctx context.Context, tapeName string, entry Entry) error {
    if entry.Kind == "tool_result" {
        if content, ok := entry.Payload["content"].(string); ok {
            entry.Payload["content"] = m.redactor.Redact(content)
        }
    }
    return m.store.Append(ctx, tapeName, entry)
}
```

## 实现步骤

- [ ] 创建 `pkg/credentials/redactor.go`（Redactor 核心）
- [ ] 在 credentials 插件的 `BeforeToolCall` hook 中注册凭据值到 Redactor
- [ ] 在 `pkg/tool/executor.go` 执行结果返回前调用 `Redact()`
- [ ] 在 `pkg/tape/manager.go` 写入时做二次脱敏
- [ ] 更新默认 OPA 策略，拒绝 env dump 命令
- [ ] 添加单元测试（含多凭据、嵌套值、短值跳过等场景）
- [ ] 添加 `[credentials] redaction_enabled = true` 配置项（默认开启）

## 代价与风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| 误匹配：凭据值恰好出现在正常文本中 | 低 | 中 | 最小长度阈值（4 字符），placeholder 包含凭据名便于排查 |
| 性能：每次工具输出都做字符串扫描 | 中 | 低 | 只在有注册凭据时扫描；大多数场景凭据数 < 10 |
| 编码绕过：凭据值经 base64/URL 编码后不匹配 | 中 | 中 | 对常见编码变体（base64、URL encode）也注册匹配 |
| Redactor 注册的凭据在内存中是明文 | 确定 | 低 | 与 os.Setenv 同等风险，进程结束后清除 |

## 参考

- DMR `plugins/credentials/` — 凭据存储和 binding 注入
- DMR `pkg/tool/executor.go` — 工具执行和结果处理
- DMR `docs/issues/secret-redaction-via-credential-match.md` — 主仓库中的初步设计
- AWS CLI — 类似的 `--no-cli-pager` + credential redaction 机制
