# 内置插件实现分析

## 概述

DMR 提供多个内置插件，每个插件实现特定功能。

## 插件清单

| 插件 | 工具 | 钩子 | 用途 |
|------|------|------|------|
| `shell` | `shell`, `shellOutput`, `shellKill`, `shellJobs` | RegisterCoreTools | 执行 shell 命令 |
| `fs` | `fsRead`, `fsWrite`, `fsEdit` | RegisterCoreTools | 文件系统操作 |
| `tape` | `tapeSearch`, `tapeHandoff`, `tapeAnchors`, `tapeInfo` | RegisterCoreTools | Tape 操作 |
| `subagent` | `subagent` | RegisterCoreTools | 子 Agent |
| `skill` | `skill` | RegisterExtendedTools, SystemPrompt | 技能加载 |
| `clawhub` | `clawhubSearch`, `clawhubInstall`, `clawhubRemove`, `clawhubList` | RegisterExtendedTools | ClawHub 技能市场 |
| `webtool` | `webFetch`, `webSearch` | RegisterExtendedTools | Web 访问 |
| `toolsearch` | `toolSearch` | RegisterCoreTools | 工具搜索 |
| `credentials` | `credentialEnv` | RegisterCoreTools | 凭证管理 |
| `powershell` | `powershell`, `powershellOutput` | RegisterCoreTools | PowerShell (Windows) |
| `command` | - | InterceptInput | 命令拦截 |
| `opa_policy` | - | BeforeToolCall, BatchBeforeToolCall | 策略引擎 |
| `cli` | - | ProvideApprover, ProvideChatInterface | CLI 界面 |
| `osinfo` | - | SystemPrompt | 系统信息 |
| `cron` | - | SetAgentRunner, SetTapeControl | 定时任务 |
| `webserver` | - | ProvideApprover, SetAgentRunner, SetTapeReader, SetTapeControl, AfterAgentRun | Web 界面 |
| `mcp` | MCP tools | RegisterExtendedTools | MCP 服务器桥接 |

---

## 1. Shell 插件

### 功能
- 执行 shell 命令（同步/后台）
- 读取后台命令输出
- 终止后台命令
- 列出后台作业

### 核心实现

```go
type ShellPlugin struct {
    shellManager   *ShellManager
    interactive    bool  // -i 标志
    defaultTimeout int   // 默认超时秒数
}

type ShellManager struct {
    mu     sync.Mutex
    shells map[string]*Shell
}

type Shell struct {
    ID         string
    Cmd        string
    Cwd        string
    OutputBuf  *bytes.Buffer
    Process    *os.Process
    Status     string  // running, completed, terminated
    ReturnCode *int
}
```

### shell 工具

```go
func (p *ShellPlugin) shellTool() *tool.Tool {
    return &tool.Tool{
        Spec: tool.ToolSpec{
            Name:        "shell",
            Description: "Execute a shell command",
            Parameters: map[string]any{
                "type": "object",
                "properties": map[string]any{
                    "cmd":             map[string]any{"type": "string"},
                    "cwd":             map[string]any{"type": "string"},
                    "timeout_seconds": map[string]any{"type": "integer", "default": 30},
                    "background":      map[string]any{"type": "boolean", "default": false},
                    "credential_bindings": {...},
                },
                "required": []string{"cmd"},
            },
        },
        Handler:     p.execHandler,
        NeedContext: true,
    }
}
```

### 执行流程

```go
func (p *ShellPlugin) execHandler(ctx *tool.ToolContext, args map[string]any) (any, error) {
    cmd := args["cmd"].(string)
    cwd := p.determineCwd(ctx, args["cwd"].(string))
    timeout := p.defaultTimeout
    background := args["background"].(bool)
    
    // CWD 管理
    if err := ctx.CheckCwdAllowed(cwd); err != nil {
        return nil, fmt.Errorf("cwd not allowed: %w", err)
    }
    
    // 后台模式
    if background {
        shellID := p.shellManager.Start(cmd, cwd, p.interactive, extraEnv, tempFiles)
        return fmt.Sprintf("started: %s", shellID), nil
    }
    
    // 同步执行
    output, exitCode, err := executeCommand(cmd, cwd, timeout, p.interactive, extraEnv, tempFiles)
    
    // CWD 跟踪
    if ctx.CwdPolicy == tool.CwdPolicyTrack && isCdCommand(cmd) {
        newCwd := extractCdTarget(cmd, cwd)
        ctx.CwdManager.Set(newCwd)
    }
    
    if exitCode != 0 {
        return fmt.Sprintf("%s\n(exit code: %d)", output, exitCode), nil
    }
    return output, nil
}
```

---

## 2. FS 插件

### 功能
- 读取文件（支持分页）
- 写入文件
- 编辑文件（替换文本）

### fsRead

```go
func (p *FSPlugin) readHandler(ctx *tool.ToolContext, args map[string]any) (any, error) {
    path := resolvePath(ctx, args["path"].(string))
    
    text, _ := os.ReadFile(path)
    lines := strings.Split(string(text), "\n")
    
    offset := int(args["offset"].(float64))
    limit := 0
    if l, ok := args["limit"].(float64); ok {
        limit = int(l)
    }
    
    end := len(lines)
    if limit > 0 && offset+limit < end {
        end = offset + limit
    }
    
    return strings.Join(lines[offset:end], "\n"), nil
}
```

### fsEdit

```go
func (p *FSPlugin) editHandler(ctx *tool.ToolContext, args map[string]any) (any, error) {
    path := resolvePath(ctx, args["path"].(string))
    old := args["old"].(string)
    newText := args["new"].(string)
    start := int(args["start"].(float64))
    
    text, _ := os.ReadFile(path)
    lines := strings.Split(string(text), "\n")
    
    // 只替换 start 行之后的内容
    prev := strings.Join(lines[:start], "\n")
    toReplace := strings.Join(lines[start:], "\n")
    
    if !strings.Contains(toReplace, old) {
        return nil, fmt.Errorf("'%s' not found", old)
    }
    
    replaced := strings.Replace(toReplace, old, newText, 1)
    if prev != "" {
        replaced = prev + "\n" + replaced
    }
    
    os.WriteFile(path, []byte(replaced), 0644)
    return fmt.Sprintf("edited: %s", path), nil
}
```

---

## 3. Tape 插件

### 功能
- 搜索历史记录 (FTS5/tsvector)
- 创建 handoff anchor
- 列出 anchors
- 获取 tape 信息

### tapeSearch

```go
func (p *TapePlugin) searchHandler(ctx *tool.ToolContext, args map[string]any) (any, error) {
    store := ctx.State["_tape_store"].(tape.TapeStore)
    
    opts := &tape.FetchOpts{
        TextQuery: args["query"].(string),
        Kinds:     args["kinds"].([]string),
        Limit:     int(args["limit"].(float64)),
        StartDate: args["start_date"].(string),
        EndDate:   args["end_date"].(string),
        AfterID:   int(args["after_id"].(float64)),
    }
    
    entries, _ := store.FetchAll(ctx.Tape, opts)
    
    // 处理结果，支持 role 过滤
    var lines []string
    for _, entry := range entries {
        if roleFilter != "" && entry.Kind == "message" {
            if role, ok := entry.Payload["role"].(string); ok && role != roleFilter {
                continue
            }
        }
        // 格式化为 JSON 行
        lines = append(lines, formatEntry(entry))
    }
    
    return map[string]any{
        "matches":     len(lines),
        "next_cursor": nextCursor,
        "entries":     strings.Join(lines, "\n"),
    }, nil
}
```

---

## 4. OPA Policy 插件

### 功能
- 使用 Open Policy Agent 评估工具调用
- 支持 allow/deny/require_approval 决策
- 批量策略检查
- 策略热重载

### 核心结构

```go
type OPAPolicyPlugin struct {
    engine       *rego.PreparedEvalQuery
    engineMu     sync.RWMutex
    cache        *ApprovalCache
    audit        *AuditLogger
    reloadConfig map[string]any
    reloadStop   func()
}
```

### 策略评估

```go
func (p *OPAPolicyPlugin) beforeToolCallTyped(ctx context.Context, args plugin.BeforeToolCallArgs) error {
    input := map[string]any{
        "tool": t.Spec.Name,
        "args": toolArgs,
        "context": map[string]any{
            "tape":      toolCtx.Tape,
            "workspace": toolCtx.State["_runtime_workspace"],
        },
    }
    
    decision, _ := p.evaluate(ctx, input)
    
    switch decision.Action {
    case "allow":
        return nil
    case "deny":
        return fmt.Errorf("denied by policy: %s", decision.Reason)
    case "require_approval":
        return p.handleApproval(ctx, decision, t.Spec.Name, toolArgs, toolCtx.Tape)
    }
}
```

### 决策流程

```go
func (p *OPAPolicyPlugin) evaluate(ctx context.Context, input map[string]any) (plugin.Decision, error) {
    p.engineMu.RLock()
    eng := p.engine
    p.engineMu.RUnlock()
    
    results, _ := eng.Eval(ctx, rego.EvalInput(input))
    
    raw := results[0].Expressions[0].Value.(map[string]any)
    return plugin.Decision{
        Action: raw["action"].(string),
        Reason: raw["reason"].(string),
        Risk:   raw["risk"].(string),
    }, nil
}
```

---

## 5. Skill 插件

### 功能
- 从 SKILL.md 文件加载技能
- 动态系统提示贡献
- 支持 skills/local 目录

### 技能文件格式

```markdown
---
name: docker-compose
description: Manage Docker Compose services
---

# Docker Compose Skill

## Usage

When working with Docker Compose:
1. Check `docker-compose.yml` exists
2. Use `docker compose ps` to check status
3. ...
```

### 实现

```go
type SkillPlugin struct {
    skills        []*Skill
    resolvedRoots []string
    lastScanMtime time.Time
}

type Skill struct {
    Name        string
    Description string
    Content     string
    Location    string
}

func (p *SkillPlugin) systemPrompt(_ context.Context, _ ...any) (any, error) {
    p.ensureSkillsFresh()
    if len(p.skills) == 0 {
        return "", nil
    }
    
    var lines []string
    lines = append(lines, "<available_skills>")
    for _, s := range p.skills {
        lines = append(lines, "  <skill>")
        lines = append(lines, fmt.Sprintf("    <name>%s</name>", escapeXML(s.Name)))
        lines = append(lines, fmt.Sprintf("    <description>%s</description>", escapeXML(s.Description)))
        lines = append(lines, "  </skill>")
    }
    lines = append(lines, "</available_skills>")
    return strings.Join(lines, "\n"), nil
}
```

---

## 6. Webserver 插件

### 功能
- HTTP API 服务
- Web UI 支持
- 审批界面
- SSE 实时通知

### 核心结构

```go
type Plugin struct {
    config          ServerConfig
    approver        *WebApprover
    approverAdapter *WebApproverAdapter
    server          *Server
    
    // 依赖注入
    runner     plugin.AgentRunner
    tapeReader plugin.TapeReader
    tapeCtrl   plugin.TapeControl
    
    tapeLocks sync.Map
}
```

### 依赖注入

```go
func (p *Plugin) RegisterHooks(registry *plugin.HookRegistry) {
    registry.RegisterHook(plugin.HookProvideApprover, p.Name(), 0, p.provideApprover)
    registry.RegisterSetAgentRunner(p.Name(), 0, p.setAgentRunnerTyped)
    registry.RegisterSetTapeReader(p.Name(), 0, p.setTapeReaderTyped)
    registry.RegisterSetTapeControl(p.Name(), 0, p.setTapeControlTyped)
    registry.RegisterAfterAgentRun(p.Name(), 0, p.afterAgentRunTyped)
}

func (p *Plugin) setAgentRunnerTyped(_ context.Context, runner plugin.AgentRunner) error {
    p.runner = runner
    p.wireHandlers()
    return nil
}

func (p *Plugin) wireHandlers() {
    if p.runner != nil && p.tapeReader != nil && p.tapeCtrl != nil {
        p.server.SetChatHandler(p.handleChat)
        p.server.SetHistoryHandler(p.handleHistory)
        p.server.SetTapeAPIHandlers(p.listTapes, p.createTape, p.tapeAnchors, p.tapeHandoff)
    }
}
```

---

## 7. Command 插件

### 功能
- 拦截以 `,` 开头的命令
- 实现 REPL 命令（如 `,model.switch`, `,compact`）

### 实现

```go
func (p *CommandPlugin) RegisterHooks(registry *plugin.HookRegistry) {
    registry.RegisterHook(plugin.HookInterceptInput, p.Name(), 0, p.intercept)
}

func (p *CommandPlugin) intercept(_ context.Context, args ...any) (any, error) {
    prompt := args[1].(string)
    
    if !strings.HasPrefix(prompt, ",") {
        return nil, nil  // 不是命令，继续处理
    }
    
    // 解析命令
    parts := strings.Fields(prompt[1:])
    if len(parts) == 0 {
        return &InterceptResult{Output: "Unknown command"}, nil
    }
    
    switch parts[0] {
    case "model.switch":
        return p.handleModelSwitch(parts[1:])
    case "compact":
        return p.handleCompact()
    case "restart":
        return p.handleRestart()
    // ...
    }
    
    return &InterceptResult{Output: "Unknown command: " + parts[0]}, nil
}
```

---

## 8. ClawHub 插件

### 功能
- 搜索 ClawHub 技能市场
- 安装/卸载技能
- 列出已安装技能

### API 调用

```go
func (p *ClawHubPlugin) clawhubSearchHandler(ctx *tool.ToolContext, args map[string]any) (any, error) {
    query := args["query"].(string)
    
    req, _ := http.NewRequest("GET", p.baseURL+"/api/skills/search?q="+url.QueryEscape(query), nil)
    if p.authToken != "" {
        req.Header.Set("Authorization", "Bearer "+p.authToken)
    }
    
    resp, _ := p.client.Do(req)
    body, _ := io.ReadAll(resp.Body)
    
    return string(body), nil
}
```
