# 插件系统深入分析

## 概述

DMR 采用插件优先架构，所有功能（工具、策略、技能）都实现为插件。核心只提供 LLM + Tape + Hooks。

## 插件接口

```go
type Plugin interface {
    Name() string
    Version() string
    Init(ctx context.Context, config map[string]any) error
    RegisterHooks(registry *HookRegistry)
    Shutdown(ctx context.Context) error
}
```

### 可选接口

```go
// HealthChecker 健康检查接口
type HealthChecker interface {
    HealthCheck(ctx context.Context) error
}
```

## 插件管理器 (Manager)

```go
type Manager struct {
    mu       sync.RWMutex
    plugins  []Plugin
    registry *HookRegistry
}

// 注册插件
func (m *Manager) Register(p Plugin) error

// 初始化所有插件
func (m *Manager) InitAll(ctx context.Context, configs map[string]map[string]any) error

// 关闭所有插件（逆序）
func (m *Manager) ShutdownAll(ctx context.Context) error

// 获取钩子注册表
func (m *Manager) Hooks() *HookRegistry

// 健康检查
func (m *Manager) HealthCheckAll(ctx context.Context) map[string]error
```

## Hook 系统

### Hook 名称常量

```go
const (
    HookRegisterTools         = "RegisterTools"         // 已弃用
    HookRegisterCoreTools     = "RegisterCoreTools"     // 注册核心工具
    HookRegisterExtendedTools = "RegisterExtendedTools" // 注册扩展工具
    HookSystemPrompt          = "SystemPrompt"          // 贡献系统提示片段
    HookBeforeToolCall        = "BeforeToolCall"        // 工具调用前策略检查
    HookBatchBeforeToolCall   = "BatchBeforeToolCall"   // 批量策略检查
    HookProvideApprover       = "ProvideApprover"       // 提供审批器
    HookProvideChatInterface  = "ProvideChatInterface"  // 提供聊天界面
    HookInterceptInput        = "InterceptInput"        // 拦截用户输入
    HookSetAgentRunner        = "SetAgentRunner"        // 注入 Agent 运行器
    HookSetTapeReader         = "SetTapeReader"         // 注入 Tape 读取器
    HookSetTapeControl        = "SetTapeControl"        // 注入 Tape 控制器
    HookSetHostService        = "SetHostService"        // 注入主机服务
    HookAfterAgentRun         = "AfterAgentRun"         // Agent 运行后回调
)
```

### Hook 调用策略

| 策略 | 方法 | 行为 |
|------|------|------|
| CallAll | `CallAll(ctx, hook)` | 调用所有实现，收集结果 |
| CallFirstResult | `CallFirstResult(ctx, hook)` | 遇到第一个非 nil 结果停止 |
| CallUntilError | `CallUntilError(ctx, hook)` | 遇到第一个错误停止 |

### HookRegistry

```go
type HookRegistry struct {
    mu    sync.RWMutex
    hooks map[string][]hookEntry  // hook name -> entries
}

type hookEntry struct {
    plugin   string
    priority int  // 越大越先执行
    fn       HookFunc
}
```

### 注册 Hook

```go
func (r *HookRegistry) RegisterHook(hookName, pluginName string, priority int, fn HookFunc)

// 便捷方法
func (r *HookRegistry) RegisterCoreTools(pluginName string, priority int, fn HookFunc)
func (r *HookRegistry) RegisterExtendedTools(pluginName string, priority int, fn HookFunc)
func (r *HookRegistry) RegisterSystemPrompt(pluginName string, priority int, fn HookFunc)
```

## 已注册的 Hooks

| Hook 名称 | 策略 | 描述 |
|-----------|------|------|
| `RegisterCoreTools` | CallAll | 返回核心工具列表 |
| `RegisterExtendedTools` | CallAll | 返回扩展工具列表 |
| `SystemPrompt` | CallAll | 贡献系统提示片段 |
| `BeforeToolCall` | CallUntilError | 工具调用前策略检查 |
| `BatchBeforeToolCall` | CallUntilError | 批量策略检查 |
| `ProvideApprover` | CallFirstResult | 返回审批器 |
| `ProvideChatInterface` | CallFirstResult | 返回聊天界面 |
| `InterceptInput` | CallFirstResult | 拦截用户输入 |
| `SetAgentRunner` | CallAll | 注入 Agent 运行器 |
| `SetTapeReader` | CallAll | 注入 Tape 读取器 |
| `SetTapeControl` | CallAll | 注入 Tape 控制器 |
| `AfterAgentRun` | CallAll | Agent 运行后回调 |

## 配置绑定

```go
// BindConfig 将 map[string]any 绑定到结构体
func BindConfig[T any](config map[string]any, target *T) error {
    if len(config) == 0 {
        return nil
    }
    data, err := json.Marshal(config)
    if err != nil {
        return fmt.Errorf("bind config marshal: %w", err)
    }
    if err := json.Unmarshal(data, target); err != nil {
        return fmt.Errorf("bind config unmarshal: %w", err)
    }
    return nil
}

// 使用示例
type ShellConfig struct {
    Interactive bool `json:"interactive"`
    Timeout     int  `json:"timeout"`
}

var cfg ShellConfig
if err := plugin.BindConfig(config, &cfg); err != nil {
    return err
}
```

## 系统提示组合

```go
func ComposeSystemPrompt(ctx context.Context, hooks *HookRegistry, base string) string {
    if !hooks.HasHook(HookSystemPrompt) {
        return base
    }
    
    results, _ := hooks.CallAll(ctx, HookSystemPrompt)
    
    var fragments []string
    if base != "" {
        fragments = append(fragments, base)
    }
    
    for _, r := range results {
        if s, ok := r.(string); ok && s != "" {
            fragments = append(fragments, s)
        }
    }
    
    return strings.Join(fragments, "\n\n")
}
```

## 工具收集

```go
func CollectTools(results []any) []*tool.Tool {
    var tools []*tool.Tool
    for _, r := range results {
        switch v := r.(type) {
        case *tool.Tool:
            tools = append(tools, v)
        case []*tool.Tool:
            tools = append(tools, v...)
        }
    }
    return tools
}
```

## 添加新插件的步骤

1. 创建 `plugins/<name>/<name>.go`
2. 实现 `plugin.Plugin` 接口
3. 在 `RegisterHooks()` 中注册钩子
4. 在 `cmd/dmr/plugins.go:buildPluginManager()` 中添加工厂
5. 在 `pkg/config/config.go:DefaultConfig()` 中添加默认配置

```go
// 插件示例结构
package myplugin

import (
    "context"
    "github.com/seanly/dmr/pkg/plugin"
    "github.com/seanly/dmr/pkg/tool"
)

type MyPlugin struct {
    config MyConfig
}

type MyConfig struct {
    Enabled bool   `json:"enabled"`
    Option  string `json:"option"`
}

func New() plugin.Plugin { return &MyPlugin{} }

func (p *MyPlugin) Name() string    { return "myplugin" }
func (p *MyPlugin) Version() string { return "1.0.0" }

func (p *MyPlugin) Init(_ context.Context, config map[string]any) error {
    return plugin.BindConfig(config, &p.config)
}

func (p *MyPlugin) RegisterHooks(registry *plugin.HookRegistry) {
    registry.RegisterCoreTools(p.Name(), 0, p.registerTools)
}

func (p *MyPlugin) Shutdown(_ context.Context) error { return nil }

func (p *MyPlugin) registerTools(_ context.Context, _ ...any) (any, error) {
    return []*tool.Tool{p.myTool()}, nil
}

func (p *MyPlugin) myTool() *tool.Tool {
    return &tool.Tool{
        Spec: tool.ToolSpec{
            Name:        "myTool",
            Description: "My tool description",
            Parameters: map[string]any{
                "type": "object",
                "properties": map[string]any{
                    "arg": map[string]any{"type": "string"},
                },
            },
        },
        Handler: func(ctx *tool.ToolContext, args map[string]any) (any, error) {
            return "result", nil
        },
    }
}
```
