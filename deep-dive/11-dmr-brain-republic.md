# DMR Brain 与 Republic 深入分析

## 概述

- **Republic**: DMR 的 Facade API 库，提供简化的 LLM 交互接口
- **DMR Brain**: 多通道 AI Agent 编排器，支持后台服务运行

## Republic 库

### 设计目标

Republic 是一个 tape-first 的 LLM 客户端库，核心理念：
- **Decide**: 让 AI 使用工具调用做出智能决策
- **Monitor**: 在 Tape 审计日志中记录一切
- **Review**: 通过结构化数据验证 AI 决策

### 核心类型

```go
// LLM 是 Republic 的主入口
type LLM struct {
    chat      *client.ChatClient
    text      *client.TextClient
    embedding *openai.Client
    tape      *tape.TapeManager
    config    Config
    llmCore   *core.LLMCore
}

// Config 配置
 type Config struct {
    Model           string
    APIKey          string
    BaseURL         string
    FallbackModels  []string
    MaxRetries      int
    Verbose         int
    TapeStore       tape.TapeStore
    Context         *tape.TapeContext
    ErrorClassifier func(error) core.ErrorKind
    Headers         map[string]string
}
```

### API 方法

```go
// 创建实例
llm := republic.New(cfg)

// 非流式对话
response, err := llm.Chat(ctx, "Hello")

// 获取工具调用
toolCalls, err := llm.ToolCalls(ctx, "List files", republic.WithTools(tools...))

// 自动执行工具
result, err := llm.RunTools(ctx, "List files", republic.WithTools(tools...))

// 流式对话
stream, state, err := llm.Stream(ctx, "Tell me a story")
for chunk := range stream {
    fmt.Print(chunk)
}

// 流式事件
events, state, err := llm.StreamEvents(ctx, prompt, opts...)
for event := range events {
    switch event.Kind {
    case core.StreamText:
        fmt.Print(event.Data["delta"])
    case core.StreamToolCall:
        // 处理工具调用
    case core.StreamToolResult:
        // 处理工具结果
    case core.StreamUsage:
        // 处理用量
    }
}

// 条件判断
yes, err := llm.If(ctx, text, "Is this a question?")

// 分类
label, err := llm.Classify(ctx, text, []string{"positive", "negative", "neutral"})

// 嵌入向量
embedResp, err := llm.Embed(ctx, "text-embedding-3-small", []string{"text1", "text2"})

// Tape 会话
tapeSession := llm.Tape("my-session")
response, err := tapeSession.Chat(ctx, "Hello")
tapeSession.Handoff("phase2", nil)
```

### 使用示例

```go
package main

import (
    "context"
    "fmt"
    "github.com/seanly/dmr/pkg/republic"
)

func main() {
    llm := republic.New(republic.Config{
        Model:   "gpt-4o",
        APIKey:  os.Getenv("AI_API_KEY"),
        BaseURL: "https://api.openai.com/v1",
        Verbose: 1,
    })
    
    ctx := context.Background()
    
    // 简单对话
    resp, _ := llm.Chat(ctx, "What is Go?")
    fmt.Println(resp)
    
    // 使用工具
    result, _ := llm.RunTools(ctx, "List files in current directory",
        republic.WithTools(fsTools...),
    )
    fmt.Println(result.Text)
    
    // Tape 会话
    session := llm.Tape("project-alpha")
    resp, _ = session.Chat(ctx, "Start a new project")
    session.Handoff("implementation", map[string]any{"task": "setup"})
}
```

---

## DMR Brain

### 功能

DMR Brain 是一个多通道 AI Agent 编排器，支持：
- 多插件并发运行
- Web UI 和 API
- Cron 定时任务
- 外部插件集成（如 Feishu、Weixin）
- 热重载（SIGHUP）
- 系统服务管理（launchd/systemd）

### 架构

```
┌─────────────────────────────────────────────────────────────┐
│                        DMR Brain                            │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   CLI Plugin │  │ Web Plugin  │  │ Cron Plugin │         │
│  │   (disabled) │  │   (enabled) │  │  (enabled)  │         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │
│         │                │                │                 │
│         └────────────────┼────────────────┘                 │
│                          │                                  │
│                   ┌──────▼──────┐                          │
│                   │    Agent    │                          │
│                   │   (shared)  │                          │
│                   └──────┬──────┘                          │
│                          │                                  │
│                   ┌──────▼──────┐                          │
│                   │  HostService│◄── External Plugins       │
│                   │  (reverse   │    (Feishu, Weixin...)    │
│                   │   RPC)      │                          │
│                   └─────────────┘                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 启动流程

```go
func runBrainOnce() (restart bool, err error) {
    // 1. 创建 HostService（延迟绑定）
    hostService := newHostServiceForExternalPlugins(app.cfg)
    
    // 2. 构建插件管理器
    pm := buildPluginManagerWithVerbose(app.cfg, hostService, true, "cli")
    pm.InitAll(context.Background(), pluginConfigs(app.cfg))
    
    // 3. 构建 Agent
    ag, store := buildAgent(app.cfg, pm)
    tm := tape.NewTapeManager(store)
    
    // 4. 绑定 HostService
    hostService.SetRunner(&agentRunnerAdapter{ag: ag})
    hostService.SetTapeReader(&tapeReaderAdapter{tm: tm})
    hostService.SetTapeControl(&tapeControlAdapter{tm: tm})
    
    // 5. 通知插件
    pm.Hooks().CallSetAgentRunner(context.Background(), &agentRunnerAdapter{ag: ag})
    pm.Hooks().CallSetTapeReader(context.Background(), &tapeReaderAdapter{tm: tm})
    pm.Hooks().CallSetTapeControl(context.Background(), &tapeControlAdapter{tm: tm})
    pm.Hooks().CallSetHostService(context.Background(), hostService)
    
    // 6. 等待信号
    sig := <-sigCh
    
    if sig == syscall.SIGHUP {
        restart = true  // 热重载
    }
    
    // 7. 关闭
    pm.ShutdownAll(shutCtx)
    pm.KillExternal()
    
    return restart, nil
}
```

### 服务管理

```bash
# 安装服务
dmr brain service install

# 启动/停止/重启
dmr brain service start
dmr brain service stop
dmr brain service restart

# 刷新配置（发送 SIGHUP）
dmr brain service refresh

# 查看状态
dmr brain service status

# 查看日志
dmr brain service logs

# 解除卡住状态
dmr brain service unstick
```

### 服务实现

```go
func newBrainServiceCommand() *cobra.Command {
    return &cobra.Command{
        Use:   "service",
        Commands: []*cobra.Command{
            {Use: "install", Run: func(cmd *cobra.Command, args []string) {
                mgr := service.NewManager()
                mgr.Install()
            }},
            {Use: "start", Run: func(cmd *cobra.Command, args []string) {
                mgr := service.NewManager()
                mgr.Start()
            }},
            {Use: "stop", Run: func(cmd *cobra.Command, args []string) {
                mgr := service.NewManager()
                mgr.Stop()
            }},
            // ...
        },
    }
}
```

### Platform 支持

| 平台 | 服务管理 |
|------|----------|
| macOS | launchd --user |
| Linux | systemd --user |
| Windows | 不支持 |

---

## HostService 反向 RPC

### 设计

HostService 允许外部插件调用主机的功能：

```go
type HostService interface {
    // 执行 Agent 循环
    RunAgent(req *RunAgentRequest, resp *RunAgentResponse) error
    
    // 获取历史消息
    GetHistory(req *GetHistoryRequest, resp *GetHistoryResponse) error
    
    // Tape 管理
    ListTapes(req *ListTapesRequest, resp *ListTapesResponse) error
    ListTapeAnchors(req *ListTapeAnchorsRequest, resp *ListTapeAnchorsResponse) error
    CreateTape(req *CreateTapeRequest, resp *CreateTapeResponse) error
    TapeHandoff(req *TapeHandoffRequest, resp *TapeHandoffResponse) error
    AddTapeEvent(req *AddTapeEventRequest, resp *AddTapeEventResponse) error
    
    // 凭证访问
    GetCredentialSecret(req *GetCredentialSecretRequest, resp *GetCredentialSecretResponse) error
}
```

### 使用场景：Feishu 插件

```go
// Feishu 插件收到消息
func (p *FeishuPlugin) handleMessage(msg *Message) {
    req := &proto.RunAgentRequest{
        TapeName:    "feishu:" + msg.ChatID,
        Prompt:      msg.Text,
        ContextJSON: fmt.Sprintf(`{"chat_id":"%s","message_id":"%s"}`, msg.ChatID, msg.ID),
    }
    
    var resp proto.RunAgentResponse
    p.host.RunAgent(req, &resp)
    
    // 发送回复到 Feishu
    p.sendReply(msg.ChatID, resp.Output)
}

// Feishu 工具使用上下文
func (p *FeishuPlugin) replyTool(ctx *tool.ToolContext, args map[string]any) (any, error) {
    chatID := ctx.Context["chat_id"].(string)
    message := args["message"].(string)
    
    p.sendReply(chatID, message)
    return "sent", nil
}
```

---

## 适配器模式

### AgentRunnerAdapter

```go
type agentRunnerAdapter struct {
    ag *agent.Agent
}

func (a *agentRunnerAdapter) Run(ctx context.Context, tapeName, prompt string, historyAfterEntryID int32, contextJSON string) (string, int, []proto.ToolCallInfo, int, int, int, error) {
    result, err := a.ag.RunWithContext(ctx, tapeName, prompt, historyAfterEntryID, contextJSON)
    if err != nil {
        return "", 0, nil, 0, 0, 0, err
    }
    
    // 转换 ToolCallEvent 到 ToolCallInfo
    var toolCalls []proto.ToolCallInfo
    // ...
    
    return result.Output, result.Steps, toolCalls, result.PromptTokens, result.CompletionTokens, a.ag.ContextTokenBudget(tapeName), nil
}
```

### TapeReaderAdapter

```go
type tapeReaderAdapter struct {
    tm *tape.TapeManager
}

func (t *tapeReaderAdapter) ReadMessages(tape string, afterAnchor string, afterEntryID int32) ([]proto.HistoryMessage, error) {
    var ctx *tape.TapeContext
    if afterEntryID > 0 {
        entries, _ := t.tm.Store.FetchAll(tape, &tape.FetchOpts{AfterID: int(afterEntryID)})
        ctx = tape.NewNoAnchorContext()
        msgs := ctx.BuildMessages(entries)
        return convertToHistoryMessages(msgs), nil
    }
    
    if afterAnchor != "" {
        ctx = tape.NewNamedAnchorContext(afterAnchor)
    } else {
        ctx = tape.NewLastAnchorContext()
    }
    
    msgs, _ := t.tm.ReadMessages(tape, ctx)
    return convertToHistoryMessages(msgs), nil
}
```
