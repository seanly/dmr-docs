# P1: Sandbox 集成 — dmr-sandbox 沙箱执行（Sandbox Integration）

> DMR 当前 shell 插件直接在主机上执行命令，存在安全隐患。`dmr-sandbox` 项目提供跨平台沙箱库（macOS Seatbelt / Linux bubblewrap），本方案描述如何将其作为 Library 集成到 DMR，采用 "Policy-Driven Sandbox Decision" 模式实现 OPA 决策 + 沙箱执行双重保护。

## 问题

DMR 的 shell 插件（`plugins/shell/`）直接通过 `exec.Command` 在主机上执行命令：

- 命令以用户完整权限运行，无隔离
- 文件操作可以修改任意文件
- 网络访问无限制
- 凭证可能被 AI 通过输出操控泄露
- OPA policy 只能 allow/deny，无法"有限度地允许"

## DMR 的实现方案

### 架构设计

```
┌──────────────────────────────────────────────────────────┐
│  DMR Agent                                               │
│  ┌────────────────────────────────────────────────────┐  │
│  │  shell plugin                                      │  │
│  │  ┌────────────┐    ┌────────────────────────────┐  │  │
│  │  │ shell tool │───▶│ ShellExecutionStrategy     │  │  │
│  │  │  Handler   │    │  ┌──────────────────────┐  │  │  │
│  │  └────────────┘    │  │ DirectExecution      │  │  │  │
│  │                    │  └──────────────────────┘  │  │  │
│  │                    │  ┌──────────────────────┐  │  │  │
│  │                    │  │ SandboxExecution     │  │  │  │
│  │                    │  │  dmr-sandbox.Engine  │  │  │  │
│  │                    │  └──────────────────────┘  │  │  │
│  │                    └────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────┘  │
│                         │                                │
│  ┌──────────────────────┼─────────────────────────────┐  │
│  │  sandbox plugin      │                             │  │
│  │  ┌───────────────────▼─────────┐  ┌─────────────┐ │  │
│  │  │ BeforeToolCall Hook         │  │ Config      │ │  │
│  │  │  - Check if sandbox needed  │  │  - Platform │ │  │
│  │  │  - Add _sandbox directive   │  │  - Policies │ │  │
│  │  └─────────────────────────────┘  └─────────────┘ │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
                            │
                            ▼
                  ┌───────────────────────┐
                  │  dmr-sandbox library  │
                  │  ┌─────────────────┐  │
                  │  │ pkg/sandbox     │  │
                  │  │  - Engine       │  │
                  │  └─────────────────┘  │
                  │  ┌─────────────────┐  │
                  │  │ pkg/types       │  │
                  │  │  - ExecutionReq │  │
                  │  └─────────────────┘  │
                  └───────────────────────┘
```

### 执行流程

```
User Command
    │
    ▼
OPA Policy Evaluate ──► decision.sandbox = "required"/"preferred"/"disabled"
    │
    ▼
Inject _sandbox_mode to tool args
    │
    ▼
shell.execHandler()
    │
    ├──► shouldUseSandbox()
    │       Check _sandbox_mode / credentialEnv / default_mode
    │
    ├──► background + sandbox  ──► shellManager.StartSandboxed()
    ├──► background + direct   ──► shellManager.Start()
    ├──► foreground + sandbox   ──► executeInSandbox()
    └──► foreground + direct    ──► executeDirectly()
```

### Sandbox Plugin（`plugins/sandbox/sandbox.go`）

```go
type Plugin struct {
    mu      sync.RWMutex
    engine  dmrsandbox.Engine
    config  Config
}

type Config struct {
    Enabled     bool   `toml:"enabled"`
    Platform    string `toml:"platform"`     // "auto", "darwin", "linux", "noop"
    DefaultMode string `toml:"default_mode"` // "required", "preferred", "disabled"

    Filesystem FilesystemConfig `toml:"filesystem"`
    Network    NetworkConfig    `toml:"network"`
    Resources  ResourceConfig   `toml:"resources"`

    // Phase 3: 性能
    EnablePooling   bool `toml:"enable_pooling"`
    PoolSize        int  `toml:"pool_size"`
    PreheatInstance bool `toml:"preheat_instance"`

    // Phase 2: 可观测性
    AuditMode string `toml:"audit_mode"` // "tape", "log", "both", "none"
}

type FilesystemConfig struct {
    DefaultPolicy string   `toml:"default_policy"` // "allow" or "deny"
    AllowWrite    []string `toml:"allow_write"`
    ReadOnly      []string `toml:"read_only"`
}

type NetworkConfig struct {
    Mode           string   `toml:"mode"` // "allow_all", "deny_all", "filtered"
    AllowedDomains []string `toml:"allowed_domains"`
}

type ResourceConfig struct {
    MaxMemoryMB   int     `toml:"max_memory_mb"`
    MaxCPUPercent float64 `toml:"max_cpu_percent"`
    MaxFDs        int     `toml:"max_fds"`
    MaxProcesses  int     `toml:"max_processes"`
}
```

### OPA Policy 扩展

Rego 规则新增 sandbox 决策：

```rego
default sandbox_decision := "preferred"

sandbox_decision := "required" if {
    input.tool == "shell"
    is_high_risk_command(input.args.cmd)
}

sandbox_decision := "required" if {
    input.args.credentialEnv != null
    count(input.args.credentialEnv) > 0
}

sandbox_decision := "disabled" if {
    input.tool == "shell"
    simple_command(shell_cmd)  # ls, cat, echo 等
}

is_high_risk_command(cmd) if {
    dangerous_patterns := ["curl.*\\|", "wget.*\\|", "nc ", "ncat ", "openssl.*s_client"]
    pattern := dangerous_patterns[_]
    regex.match(pattern, cmd)
}
```

OPA Plugin 注入 `_sandbox_mode` 到工具参数：

```go
func (p *OPAPolicyPlugin) beforeToolCall(ctx context.Context, args ...any) (any, error) {
    decision, err := p.evaluate(ctx, input)

    sandboxMode := p.extractSandboxMode(decision)
    if sandboxMode != "" {
        if toolArgs, ok := args[1].(map[string]any); ok {
            toolArgs["_sandbox_mode"] = sandboxMode
        }
    }
    // ... 现有 allow/deny 处理 ...
}
```

### Shell Plugin 四路分叉执行

```go
func (p *ShellPlugin) execHandler(ctx *tool.ToolContext, args map[string]any) (any, error) {
    cmd, _ := args["cmd"].(string)
    cwd := p.determineCwd(ctx, args["cwd"])
    timeout := p.defaultTimeout
    background, _ := args["background"].(bool)
    extraEnv, tempFiles := extractRuntimeInject(args)

    useSandbox := p.shouldUseSandbox(args)

    if background && useSandbox {
        shellID := p.shellManager.StartSandboxed(cmd, cwd, extraEnv, timeout, p.sandboxPlugin)
        return fmt.Sprintf("started (sandboxed): %s", shellID), nil
    }
    if background {
        shellID := p.shellManager.Start(cmd, cwd, p.interactive, extraEnv, tempFiles)
        return fmt.Sprintf("started: %s", shellID), nil
    }
    if useSandbox {
        return p.executeInSandbox(cmd, cwd, timeout, extraEnv)
    }
    return p.executeDirectly(cmd, cwd, timeout, extraEnv, tempFiles)
}

func (p *ShellPlugin) shouldUseSandbox(args map[string]any) bool {
    if p.sandboxPlugin == nil || !p.sandboxPlugin.IsEnabled() {
        return false
    }
    if sandboxMode, ok := args["_sandbox_mode"].(string); ok {
        switch sandboxMode {
        case "required":  return true
        case "disabled":  return false
        case "preferred": return true
        }
    }
    if credentialEnv, ok := args["credentialEnv"].([]any); ok && len(credentialEnv) > 0 {
        return true
    }
    return p.sandboxPlugin.GetDefaultMode() != "disabled"
}

func (p *ShellPlugin) executeInSandbox(cmd, cwd string, timeout int, extraEnv map[string]string) (any, error) {
    engine := p.sandboxPlugin.GetEngine()
    req := p.sandboxPlugin.BuildExecutionRequest(getShell(), shellArgs(p.interactive, cmd), cwd, extraEnv, timeout)

    result, err := engine.Execute(context.Background(), req)
    if err != nil {
        return nil, fmt.Errorf("sandbox execution failed: %w", err)
    }

    output := result.Output.Stdout
    if result.Output.Stderr != "" {
        if output != "" { output += "\n" }
        output += result.Output.Stderr
    }
    for _, v := range result.Violations {
        // record to tape or audit log
    }
    if result.Execution.ExitCode != 0 {
        return fmt.Sprintf("%s\n(exit code: %d)", output, result.Execution.ExitCode), nil
    }
    if output == "" { return "(no output)", nil }
    return output, nil
}
```

### ShellManager 后台沙箱任务

```go
func (m *ShellManager) StartSandboxed(cmd, cwd string, extraEnv map[string]string, timeout int, sp *sandbox.Plugin) string {
    m.mu.Lock()
    m.nextID++
    id := fmt.Sprintf("sb-%08d", m.nextID)
    m.mu.Unlock()

    s := &sandboxedShell{
        shell: &shell{ID: id, status: shellRunning, done: make(chan struct{})},
        req:   sp.BuildExecutionRequest(getShell(), shellArgs(true, cmd), cwd, extraEnv, timeout),
    }

    go func() {
        defer close(s.done)
        result, err := sp.GetEngine().Execute(context.Background(), s.req)
        s.mu.Lock()
        defer s.mu.Unlock()
        if err != nil {
            s.output.WriteString(fmt.Sprintf("sandbox error: %v\n", err))
            code := -1; s.returnCode = &code
        } else {
            s.output.WriteString(result.Output.Stdout)
            if result.Output.Stderr != "" { s.output.WriteString("\n" + result.Output.Stderr) }
            s.returnCode = &result.Execution.ExitCode
        }
        if s.status == shellRunning { s.status = shellCompleted }
    }()

    m.mu.Lock()
    m.shells[id] = s.shell
    m.mu.Unlock()
    return id
}
```

### Plugin 注册与跨插件注入

```go
// cmd/dmr/commands.go
builtins := map[string]func() plugin.Plugin{
    "sandbox": func() plugin.Plugin { return sandbox.New() },
    // ...
}

// 初始化后注入
if sp, ok := pm.Get("sandbox").(*sandbox.Plugin); ok {
    if sh, ok := pm.Get("shell").(*execplugin.ShellPlugin); ok {
        sh.SetSandboxPlugin(sp)
    }
}
```

**前置条件：** `plugin.Manager` 需添加 `Get(name string) Plugin` 方法。

### 配置示例

```toml
[[plugins]]
name = "sandbox"
enabled = true

[plugins.config]
platform = "auto"
default_mode = "preferred"
enable_pooling = true
pool_size = 2
audit_mode = "tape"

[plugins.config.filesystem]
default_policy = "deny"
allow_write = ["/tmp", "/var/tmp", "/home/user/workspace"]
read_only = ["/usr", "/bin", "/lib", "/lib64"]

[plugins.config.network]
mode = "filtered"
allowed_domains = ["github.com", "go.dev", "proxy.golang.org"]

[plugins.config.resources]
max_memory_mb = 512
max_cpu_percent = 50.0
max_fds = 1024
max_processes = 32
```

### 与 Claude Code Sandbox 对比

| 特性 | Claude Code | DMR Sandbox |
|------|-------------|-------------|
| 集成方式 | TypeScript 内部实现 | Go Library (dmr-sandbox) |
| 沙箱引擎 | 未公开 | bubblewrap (Linux) + Seatbelt (macOS) |
| 决策逻辑 | `shouldUseSandbox()` 函数 | OPA policy + `BeforeToolCall` Hook |
| 权限系统 | allow/ask/deny 规则 | OPA policy + sandbox 双重保护 |
| 网络控制 | 支持（MITM 代理） | 支持（filtered mode） |
| 资源限制 | 支持 | 支持（memory/cpu/fd） |

### 文件变更清单

新建文件：

```
plugins/sandbox/
├── sandbox.go          # 主插件实现
├── sandbox_stub.go     # Windows stub
├── job_tracker.go      # 后台作业管理
└── pool.go             # 实例池化
```

修改文件：

```
pkg/plugin/manager.go              # 添加 Get() 方法
plugins/opapolicy/opapolicy.go     # Sandbox 决策注入
plugins/opapolicy/policies/default.rego  # Sandbox 规则
plugins/shell/shell.go             # 沙箱执行路径
plugins/shell/shell_manager.go     # 后台沙箱任务
cmd/dmr/commands.go                # 插件注册与注入
pkg/config/config.go               # 默认配置
go.mod                             # dmr-sandbox 依赖
config.example.toml                # 配置示例
```

## 实现步骤

### Phase 1: 基础集成 (MVP)

- [ ] 添加 `go.mod` replace 依赖 `dmr-sandbox`
- [ ] `pkg/plugin/manager.go` 添加 `Get()` 方法
- [ ] 创建 `plugins/sandbox/sandbox.go`（Plugin 接口 + Engine + BuildExecutionRequest）
- [ ] `plugins/opapolicy/opapolicy.go` 添加 `extractSandboxMode` + `_sandbox_mode` 注入
- [ ] `plugins/opapolicy/policies/default.rego` 添加 sandbox 决策规则
- [ ] 修改 `plugins/shell/shell.go` 四路分叉执行
- [ ] `cmd/dmr/commands.go` 注册 sandbox 插件 + 跨插件注入
- [ ] 更新 `pkg/config/config.go` 默认配置

### Phase 2: 高级功能

- [ ] `plugins/shell/shell_manager.go` 添加 `StartSandboxed` 后台沙箱任务
- [ ] `plugins/sandbox/job_tracker.go` 后台作业跟踪器
- [ ] `audit_mode` 配置，Violation 记录到 tape/日志
- [ ] 自定义文件系统、网络、资源策略

### Phase 3: 性能优化

- [ ] `plugins/sandbox/pool.go` 沙箱实例池化/复用
- [ ] 预热机制（`preheat_instance`）
- [ ] `plugins/sandbox/sandbox_stub.go` Windows stub

## 解决的问题

- **执行隔离**：命令在受限环境中运行，无法越权访问文件或网络
- **凭证保护**：使用 `credentialEnv` 时强制沙箱，防止凭证泄露
- **多层防御**：OPA policy（决策层）+ sandbox（执行层）双重保护
- **透明集成**：用户无感知，shell 命令自动按策略选择执行方式
- **跨平台**：macOS（Seatbelt）和 Linux（bubblewrap）原生支持
- **四路覆盖**：前台/后台 x 直接/沙箱全路径覆盖

## 代价与风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| bubblewrap/seatbelt 未安装 | 中 | 高 | 优雅降级到直接执行，记录警告 |
| 沙箱性能开销 | 中 | 中 | 实例池化复用；简单命令跳过沙箱 |
| 文件系统权限过严 | 中 | 中 | 可配置策略；调试模式 |
| 沙箱逃逸 | 低 | 高 | 及时更新底层引擎；多层防御 |
| 凭证注入路径与沙箱冲突 | 低 | 中 | 沙箱挂载包含注入的临时文件路径 |
| `plugin.Manager.Get()` 引入耦合 | 中 | 低 | 仅在 `commands.go` 初始化阶段使用 |
| Windows 不支持 | — | 低 | stub 实现，返回明确错误 |

## 参考

- DMR `plugins/shell/shell.go:119-203` — Shell 执行流程
- DMR `plugins/shell/shell_manager.go:42-87` — ShellManager
- DMR `plugins/opapolicy/opapolicy.go:111-162` — OPA 决策流程
- DMR `pkg/plugin/hooks.go` — Hook 系统
- DMR `pkg/tool/executor.go:377-404` — ToolExecutor Hook 绑定
- DMR `cmd/dmr/commands.go:166-264` — Plugin 注册
- dmr-sandbox `pkg/sandbox/` — Engine 接口
- dmr-sandbox `pkg/types/` — ExecutionRequest/Result 类型
