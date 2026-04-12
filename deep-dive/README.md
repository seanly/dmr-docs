# DMR 深入分析文档

本目录包含 DMR 项目的深入技术分析文档。

## 文档索引

| 文档 | 内容 |
|------|------|
| [01-overview.md](01-overview.md) | DMR 项目概览、架构图、数据流 |
| [02-tape-system.md](02-tape-system.md) | Tape 审计日志系统设计、存储实现、Handoff 机制 |
| [03-llm-core.md](03-llm-core.md) | LLM 执行引擎、重试与降级策略、错误分类 |
| [04-plugin-system.md](04-plugin-system.md) | 插件系统架构、Hook 机制、插件生命周期 |
| [05-agent-loop.md](05-agent-loop.md) | Agent 执行循环、上下文管理、自动压缩 |
| [06-tool-system.md](06-tool-system.md) | 工具系统、ToolContext、并行执行 |
| [07-config-system.md](07-config-system.md) | 配置系统、多模型支持、TOML 解析 |
| [08-external-plugins.md](08-external-plugins.md) | 外部插件机制、RPC 协议、Feishu 插件示例 |
| [09-startup-flow.md](09-startup-flow.md) | 启动流程、初始化顺序、关闭流程 |
| [10-builtin-plugins.md](10-builtin-plugins.md) | 内置插件实现分析 |
| [11-dmr-brain-republic.md](11-dmr-brain-republic.md) | DMR Brain 和 Republic 库设计 |
| [12-project-structure.md](12-project-structure.md) | 项目目录结构、关键文件说明 |

## 快速导航

### 核心包

| 包 | 路径 | 说明 |
|----|------|------|
| `pkg/agent` | [pkg/agent/](pkg/agent/) | Agent 循环 |
| `pkg/client` | [pkg/client/](pkg/client/) | LLM 客户端 |
| `pkg/config` | [pkg/config/](pkg/config/) | 配置管理 |
| `pkg/core` | [pkg/core/](pkg/core/) | 执行引擎 |
| `pkg/openai` | [pkg/openai/](pkg/openai/) | OpenAI 客户端 |
| `pkg/plugin` | [pkg/plugin/](pkg/plugin/) | 插件系统 |
| `pkg/republic` | [pkg/republic/](pkg/republic/) | Facade API |
| `pkg/tape` | [pkg/tape/](pkg/tape/) | 审计日志 |
| `pkg/tool` | [pkg/tool/](pkg/tool/) | 工具系统 |

### 内置插件

| 插件 | 路径 | 说明 |
|------|------|------|
| `cli` | [plugins/cli/](plugins/cli/) | CLI 界面 |
| `command` | [plugins/command/](plugins/command/) | 命令拦截 |
| `cron` | [plugins/cron/](plugins/cron/) | 定时任务 |
| `fs` | [plugins/fs/](plugins/fs/) | 文件系统 |
| `mcp` | [plugins/mcp/](plugins/mcp/) | MCP 桥接 |
| `opa_policy` | [plugins/opapolicy/](plugins/opapolicy/) | OPA 策略 |
| `shell` | [plugins/shell/](plugins/shell/) | Shell 执行 |
| `skill` | [plugins/skill/](plugins/skill/) | 技能系统 |
| `tape` | [plugins/tape/](plugins/tape/) | Tape 工具 |
| `webserver` | [plugins/webserver/](plugins/webserver/) | Web 服务 |
| `webtool` | [plugins/webtool/](plugins/webtool/) | Web 工具 |

## 核心概念

### 1. Tape (审计日志)

Tape 是 DMR 的核心创新，所有对话和工具调用都被记录为不可变的结构化数据。

```
TapeEntry:
  - ID: 唯一标识
  - Kind: 类型 (message, tool_call, anchor, event...)
  - Payload: 数据载荷
  - Date: 时间戳
```

### 2. Agent Loop

多轮对话循环：LLM 调用 → 工具执行 → 记录结果 → 下一轮。

```
for step := 1; step <= maxSteps; step++ {
    response := llm.Call(messages, tools)
    if response.HasToolCalls() {
        results := executeTools(response.ToolCalls)
        messages = append(messages, results)
    } else {
        return response.Text
    }
}
```

### 3. Plugin System

所有功能都是插件，通过 Hook 机制集成。

```go
type Plugin interface {
    Name() string
    Version() string
    Init(ctx context.Context, config map[string]any) error
    RegisterHooks(registry *HookRegistry)
    Shutdown(ctx context.Context) error
}
```

### 4. Tool System

工具定义 + 执行器 + 上下文。

```go
type Tool struct {
    Spec    ToolSpec
    Handler func(ctx *ToolContext, args map[string]any) (any, error)
}
```

### 5. OPA Policy

使用 Open Policy Agent 评估每个工具调用。

```rego
allow if {
    input.tool == "fsRead"
    startswith(input.args.path, input.context.workspace)
}
```

## 数据流图

```
┌─────────┐    ┌──────────┐    ┌─────────┐    ┌─────────┐
│  User   │───▶│  Agent   │───▶│  LLM    │───▶│ Response│
└─────────┘    └────┬─────┘    └─────────┘    └────┬────┘
                    │                                │
                    ▼                                ▼
              ┌──────────┐                    ┌──────────┐
              │  Tape    │◀───────────────────│ ToolExec │
              │ (record) │                    │ (policy) │
              └──────────┘                    └────┬─────┘
                                                   │
                              ┌────────────────────┼────────────────────┐
                              ▼                    ▼                    ▼
                         ┌─────────┐        ┌─────────┐          ┌─────────┐
                         │  shell  │        │   fs    │          │  other  │
                         └─────────┘        └─────────┘          └─────────┘
```

## 扩展阅读

- [项目 README](../../README.md)
- [AGENTS.md](../../AGENTS.md) - 开发指南
- [docs/](../../docs/) - 设计文档
