# 外部插件机制深入分析

## 概述

DMR 支持通过 HashiCorp go-plugin 加载外部插件，实现进程隔离和动态扩展。

## 架构

```
┌─────────────────────────────────────────────────────────────┐
│                     DMR Host Process                        │
│  ┌─────────────────┐         ┌─────────────────────────┐   │
│  │ Plugin Manager  │◄────────│   ExternalPlugin (RPC)  │   │
│  │                 │         │  - ExternalApprover     │   │
│  │ - Register()    │         │  - registerTools()      │   │
│  │ - InitAll()     │         │  - Shutdown()           │   │
│  └─────────────────┘         └───────────┬─────────────┘   │
│           │                               │                 │
│  ┌────────▼────────┐          ┌───────────▼────────────┐   │
│  │ AgentHostService│          │   External Plugin      │   │
│  │ - RunAgent()    │◄────────►│   (Separate Process)   │   │
│  │ - GetHistory()  │   RPC    │                        │   │
│  │ - ListTapes()   │          │  ┌──────────────────┐  │   │
│  └─────────────────┘          │  │ DMRPluginServer  │  │   │
│                               │  │ - Init()         │  │   │
│                               │  │ - ProvideTools() │  │   │
│                               │  │ - CallTool()     │  │   │
│                               │  │ - RequestApproval│  │   │
│                               │  └──────────────────┘  │   │
│                               └────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## 外部插件配置

```toml
[[plugins]]
name = "feishu"
enabled = true
path = "/path/to/dmr-plugin-feishu"  # 可执行文件路径
[plugins.config]
app_id = "cli_xxx"
app_secret = "xxx"
encrypt_key = "xxx"
verification_token = "xxx"
```

## 外部插件接口

### DMRPluginInterface (Host → Plugin)

```go
type DMRPluginInterface interface {
    Init(req *InitRequest, resp *InitResponse) error
    Shutdown(req *ShutdownRequest, resp *ShutdownResponse) error
    RequestApproval(req *ApprovalRequest, resp *ApprovalResult) error
    RequestBatchApproval(req *BatchApprovalRequest, resp *BatchApprovalResult) error
    ProvideTools(req *ProvideToolsRequest, resp *ProvideToolsResponse) error
    CallTool(req *CallToolRequest, resp *CallToolResponse) error
}
```

### HostService (Plugin → Host)

```go
type HostService interface {
    RunAgent(req *RunAgentRequest, resp *RunAgentResponse) error
    GetHistory(req *GetHistoryRequest, resp *GetHistoryResponse) error
    ListTapes(req *ListTapesRequest, resp *ListTapesResponse) error
    ListTapeAnchors(req *ListTapeAnchorsRequest, resp *ListTapeAnchorsResponse) error
    CreateTape(req *CreateTapeRequest, resp *CreateTapeResponse) error
    TapeHandoff(req *TapeHandoffRequest, resp *TapeHandoffResponse) error
    AddTapeEvent(req *AddTapeEventRequest, resp *AddTapeEventResponse) error
    GetCredentialSecret(req *GetCredentialSecretRequest, resp *GetCredentialSecretResponse) error
}
```

## 加载流程

```go
func LoadExternalPlugins(mgr *Manager, configs []ExternalPluginConfig, hostService proto.HostService) error {
    for _, cfg := range configs {
        if !cfg.Enabled || cfg.Path == "" {
            continue
        }
        
        path := expandPath(cfg.Path)
        
        // 创建 go-plugin 客户端
        client := goplugin.NewClient(&goplugin.ClientConfig{
            HandshakeConfig: proto.Handshake,
            Plugins: map[string]goplugin.Plugin{
                "dmr-plugin": &proto.DMRPlugin{HostImpl: hostService},
            },
            Cmd:    exec.Command(path),
            Logger: logger,
        })
        
        // 连接 RPC
        rpcClient, err := client.Client()
        if err != nil {
            client.Kill()
            return fmt.Errorf("connect to plugin %s: %w", cfg.Name, err)
        }
        
        // 获取插件接口
        raw, err := rpcClient.Dispense("dmr-plugin")
        if err != nil {
            client.Kill()
            return fmt.Errorf("dispense plugin %s: %w", cfg.Name, err)
        }
        
        rpcImpl := raw.(proto.DMRPluginInterface)
        
        // 包装为 ExternalPlugin
        ep := &ExternalPlugin{
            rpc:  rpcImpl,
            name: cfg.Name,
            kill: client.Kill,
        }
        
        // 注册到管理器
        mgr.Register(ep)
    }
    return nil
}
```

## ExternalPlugin 实现

```go
type ExternalPlugin struct {
    rpc  proto.DMRPluginInterface
    name string
    kill func()  // 终止插件进程
}

func (p *ExternalPlugin) Name() string    { return p.name }
func (p *ExternalPlugin) Version() string { return "external" }

func (p *ExternalPlugin) Init(_ context.Context, config map[string]any) error {
    configJSON, _ := json.Marshal(config)
    return p.rpc.Init(&proto.InitRequest{ConfigJSON: string(configJSON)}, &proto.InitResponse{})
}

func (p *ExternalPlugin) RegisterHooks(registry *HookRegistry) {
    // 注册为扩展工具提供者
    registry.RegisterHook(HookRegisterExtendedTools, p.name, 0, p.registerTools)
    
    // 注册审批器（如果插件支持）
    registry.RegisterHook("ProvideApprover", p.name, 0, func(_ context.Context, _ ...any) (any, error) {
        return &ExternalApprover{rpc: p.rpc, pluginName: p.name}, nil
    })
}

func (p *ExternalPlugin) Shutdown(ctx context.Context) error {
    // RPC 调用 Shutdown
    errCh := make(chan error, 1)
    go func() {
        errCh <- p.rpc.Shutdown(&proto.ShutdownRequest{}, &proto.ShutdownResponse{})
    }()
    
    select {
    case <-ctx.Done():
        return ctx.Err()
    case err := <-errCh:
        if p.kill != nil {
            p.kill()
        }
        return err
    }
}
```

## 工具提供

```go
func (p *ExternalPlugin) registerTools(_ context.Context, _ ...any) (any, error) {
    var resp proto.ProvideToolsResponse
    if err := p.rpc.ProvideTools(&proto.ProvideToolsRequest{}, &resp); err != nil {
        return nil, err
    }
    
    var tools []*tool.Tool
    for _, td := range resp.Tools {
        t := &tool.Tool{
            Spec: tool.ToolSpec{
                Name:        td.Name,
                Description: td.Description,
                Parameters:  parseParameters(td.ParametersJSON),
                Group:       tool.ToolGroup(td.Group),
                SearchHint:  td.SearchHint,
                AlwaysLoad:  td.AlwaysLoad,
            },
            Handler: func(ctx *tool.ToolContext, args map[string]any) (any, error) {
                argsJSON, _ := json.Marshal(args)
                
                req := &proto.CallToolRequest{
                    Name:        td.Name,
                    ArgsJSON:    string(argsJSON),
                    SessionTape: ctx.Tape,
                    ContextJSON: encodeContext(ctx.Context),
                }
                
                var resp proto.CallToolResponse
                if err := p.rpc.CallTool(req, &resp); err != nil {
                    return nil, err
                }
                if resp.Error != "" {
                    return nil, fmt.Errorf(resp.Error)
                }
                
                var result any
                json.Unmarshal([]byte(resp.ResultJSON), &result)
                return result, nil
            },
        }
        tools = append(tools, t)
    }
    return tools, nil
}
```

## 审批适配器

```go
type ExternalApprover struct {
    rpc        proto.DMRPluginInterface
    pluginName string
}

// CanHandleTape 基于 tape 前缀路由
func (a *ExternalApprover) CanHandleTape(tape string) bool {
    return strings.HasPrefix(tape, a.pluginName+":")
}

func (a *ExternalApprover) RequestApproval(_ context.Context, req ApprovalRequest) (ApprovalResult, error) {
    argsJSON, _ := json.Marshal(req.Args)
    
    protoReq := &proto.ApprovalRequest{
        Tool:     req.Tool,
        ArgsJSON: string(argsJSON),
        Decision: proto.Decision{
            Action: req.Decision.Action,
            Reason: req.Decision.Reason,
            Risk:   req.Decision.Risk,
        },
        Tape: req.Tape,
    }
    
    var resp proto.ApprovalResult
    if err := a.rpc.RequestApproval(protoReq, &resp); err != nil {
        return ApprovalResult{Choice: Denied}, err
    }
    
    return ApprovalResult{
        Choice:  ApprovalChoice(resp.Choice),
        Comment: resp.Comment,
    }, nil
}
```

## AgentHostService

```go
type AgentHostService struct {
    mu         sync.Mutex
    runner     AgentRunner
    tapeReader TapeReader
    tapeCtrl   TapeControl
    credStore  credentials.Store
    credAllow  map[string]map[string]bool
}

func (s *AgentHostService) RunAgent(req *proto.RunAgentRequest, resp *proto.RunAgentResponse) error {
    s.mu.Lock()
    r := s.runner
    s.mu.Unlock()
    
    if r == nil {
        resp.Error = "agent not ready"
        return nil
    }
    
    output, steps, toolCalls, promptTokens, completionTokens, contextBudget, err := 
        r.Run(context.Background(), req.TapeName, req.Prompt, req.HistoryAfterEntryID, req.ContextJSON)
    
    if err != nil {
        resp.Error = err.Error()
        return nil
    }
    
    resp.Output = output
    resp.Steps = steps
    resp.ToolCalls = toolCalls
    resp.PromptTokens = promptTokens
    resp.CompletionTokens = completionTokens
    resp.ContextBudget = contextBudget
    return nil
}
```

## 上下文传递

外部插件可以通过 `ContextJSON` 传递上下文信息到工具调用：

```go
// 插件调用 RunAgent
req := &proto.RunAgentRequest{
    TapeName:    "feishu:p2p:oc_xxx",
    Prompt:      userMessage,
    ContextJSON: `{"chat_id":"oc_xxx","message_id":"om_yyy"}`,
}
host.RunAgent(req, &resp)

// 工具接收上下文
func (p *Plugin) CallTool(req *proto.CallToolRequest, resp *proto.CallToolResponse) error {
    var ctx map[string]any
    json.Unmarshal([]byte(req.ContextJSON), &ctx)
    
    chatID := ctx["chat_id"].(string)
    // 使用 chatID 无需查找 "active job"
}
```

## 凭证访问控制

```go
func (s *AgentHostService) GetCredentialSecret(req *proto.GetCredentialSecretRequest, resp *proto.GetCredentialSecretResponse) error {
    caller := req.CallerPlugin
    credID := req.CredentialID
    
    // 检查白名单
    s.mu.Lock()
    allow := s.credAllow
    s.mu.Unlock()
    
    set, ok := allow[caller]
    if !ok || len(set) == 0 {
        resp.Error = fmt.Sprintf("no credential_allowlist for plugin %q", caller)
        return nil
    }
    
    if !set[credID] {
        resp.Error = fmt.Sprintf("credential %q not allowed for plugin %q", credID, caller)
        return nil
    }
    
    // 获取凭证
    cred, err := s.credStore.Get(context.Background(), credID)
    // ... 返回凭证内容
}
```

## 配置白名单

```toml
[[plugins]]
name = "feishu"
enabled = true
path = "~/bin/dmr-plugin-feishu"
credential_allowlist = ["feishu_api_key", "webhook_secret"]
```

## 开发外部插件

### 协议定义

```go
// handshake.go
var Handshake = goplugin.HandshakeConfig{
    ProtocolVersion:  1,
    MagicCookieKey:   "DMR_PLUGIN",
    MagicCookieValue: "dmr",
}
```

### 插件实现模板

```go
package main

import (
    "github.com/hashicorp/go-plugin"
    "github.com/seanly/dmr/pkg/plugin/proto"
)

type MyPlugin struct {
    config map[string]any
}

func (p *MyPlugin) Init(req *proto.InitRequest, resp *proto.InitResponse) error {
    json.Unmarshal([]byte(req.ConfigJSON), &p.config)
    return nil
}

func (p *MyPlugin) ProvideTools(req *proto.ProvideToolsRequest, resp *proto.ProvideToolsResponse) error {
    resp.Tools = []proto.ToolDef{
        {
            Name:           "myTool",
            Description:    "Does something",
            ParametersJSON: `{"type":"object","properties":{"arg":{"type":"string"}}}`,
            Group:          "extended",
            SearchHint:     "my, tool, feature",
        },
    }
    return nil
}

func (p *MyPlugin) CallTool(req *proto.CallToolRequest, resp *proto.CallToolResponse) error {
    var args map[string]any
    json.Unmarshal([]byte(req.ArgsJSON), &args)
    
    // 执行工具逻辑
    result := doSomething(args)
    
    resultJSON, _ := json.Marshal(result)
    resp.ResultJSON = string(resultJSON)
    return nil
}

func main() {
    plugin.Serve(&plugin.ServeConfig{
        HandshakeConfig: proto.Handshake,
        Plugins: map[string]plugin.Plugin{
            "dmr-plugin": &proto.DMRPlugin{Impl: &MyPlugin{}}},
    })
}
```
