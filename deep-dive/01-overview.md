# DMR 项目概览

## 什么是 DMR

DMR (Direct, Mediate, Regulate) 是一个插件优先的 AI Agent 框架，使用 Go 语言编写。它的核心理念是：

> "In dmr we trust, but verify."

## 核心架构

```
┌─────────────────────────────────────────────────┐
│  cmd/dmr/          CLI 入口点                    │
├─────────────────────────────────────────────────┤
│  plugins/cli      交互式 REPL                   │
│  pkg/agent         Agent 循环 (LLM + 工具)       │
├─────────────────────────────────────────────────┤
│  pkg/client        高级 LLM 客户端               │
│  pkg/republic      Facade API                   │
├─────────────────────────────────────────────────┤
│  pkg/core          执行引擎                      │
│  pkg/openai        OpenAI 兼容 API 客户端       │
│  pkg/tape          追加式审计日志                │
│  pkg/tool          工具定义与执行器              │
├─────────────────────────────────────────────────┤
│  pkg/plugin        插件系统核心                  │
│  pkg/config        TOML 配置加载器               │
├─────────────────────────────────────────────────┤
│  plugins/          内置插件实现                  │
│    shell, fs, webtool, tape, subagent, cron,    │
│    skill, clawhub, opa_policy, cli, osinfo,     │
│    command, credentials, toolsearch, webserver, │
│    mcp, powershell                             │
└─────────────────────────────────────────────────┘
```

## 依赖方向

```
cmd/ → pkg/cli,agent → pkg/client → pkg/core,openai,tape,tool → pkg/plugin
```

插件只依赖于 `pkg/plugin` 和 `pkg/tool`。

## 数据流

```
用户输入
    │
    ▼
cmd/dmr ──► Agent.Run(prompt)
                │
                ▼
            ChatClient.RunTools()
                │
                ├──► LLMCore ──► OpenAI API ──► LLM Response
                │
                ▼
            Response has tool_calls?
                │
            ┌───┴───┐
            No      Yes
            │       │
            ▼       ▼
         Return   BeforeToolCall hook (OPA policy check)
         text       │
                ┌───┼───────────┐
                │   │           │
             allow  deny   require_approval
                │   │           │
                │   ▼           ▼
                │  Error    Approver.RequestApproval()
                │               │
                │           ┌───┴───┐
                │        Approved  Denied
                │           │       │
                ▼           ▼       ▼
            ToolExecutor.Execute()  Error
                │
                ▼
            Record to Tape
                │
                ▼
            Next agent step (loop back to LLM)
```

## 主要组件

| 组件 | 责任 | 关键类型 | 源文件 |
|------|------|---------|--------|
| `pkg/core` | LLM 执行引擎、错误分类、结果类型 | `LLMCore`, `LLMCoreConfig`, `LLMError`, `ExecutionResult` | `pkg/core/execution.go`, `errors.go`, `results.go` |
| `pkg/openai` | OpenAI 兼容 HTTP 客户端 | `Client`, `ChatRequest`, `ChatResponse` | `pkg/openai/client.go` |
| `pkg/tape` | 追加式对话记录 | `TapeManager`, `TapeStore`, `Entry`, `InMemoryTapeStore` | `pkg/tape/manager.go`, `store.go`, `entry.go` |
| `pkg/tool` | 工具定义、参数模式、执行 | `Tool`, `ToolExecutor`, `ToolContext` | `pkg/tool/tool.go`, `executor.go`, `context.go` |
| `pkg/client` | 高级 LLM 客户端 | `ChatClient`, `ChatOpts`, `TextClient` | `pkg/client/chat.go`, `text.go` |
| `pkg/plugin` | 插件接口、钩子注册表、审批系统 | `Plugin`, `HookRegistry`, `HookFunc`, `ApprovalCoordinator`, `Decision` | `pkg/plugin/plugin.go`, `hooks.go`, `manager.go`, `approval.go` |
| `pkg/agent` | 多轮 Agent 循环 | `Agent`, `Config`, `Result` | `pkg/agent/agent.go`, `loop.go` |
| `pkg/config` | TOML 配置加载 | `Config`, `PluginConfig` | `pkg/config/config.go` |
| `plugins/cli` | 交互式 REPL | `ChatSession`, `Renderer` | `plugins/cli/chat.go`, `renderer.go` |
| `pkg/republic` | Facade: 简单 Chat/Stream/If/Classify API | `Republic`, `Config` | `pkg/republic/republic.go` |

## 安全模型

OPA (Open Policy Agent) 策略引擎评估每个工具调用，有三种可能的结果：

- **allow** — 立即执行
- **deny** — 拒绝并说明原因
- **require_approval** — 暂停等待人工确认

审批结果有三个缓存级别：
- `ApprovedOnce` — 仅本次调用
- `ApprovedSession` — 本会话中对该工具的所有调用
- `ApprovedAlways` — 永久批准该工具
