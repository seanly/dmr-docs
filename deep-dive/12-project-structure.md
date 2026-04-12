# DMR 项目结构

## 目录概览

```
dmr/
├── cmd/dmr/                    # CLI 入口
│   ├── main.go                 # 主入口
│   ├── root.go                 # 根命令定义
│   ├── commands.go             # 子命令 (run, chat, brain)
│   ├── plugins.go              # 插件管理器构建
│   ├── agent_builder.go        # Agent 构建
│   ├── brain.go                # Brain 命令
│   ├── brain_service.go        # Brain 服务管理
│   └── ...
│
├── pkg/                        # 核心包
│   ├── agent/                  # Agent 循环
│   │   ├── agent.go            # Agent 定义和配置
│   │   ├── loop.go             # 主循环逻辑
│   │   ├── subagent.go         # 子 Agent 支持
│   │   └── ...
│   │
│   ├── client/                 # LLM 客户端
│   │   ├── chat.go             # ChatClient (RunTools)
│   │   └── text.go             # TextClient (If/Classify)
│   │
│   ├── config/                 # 配置管理
│   │   ├── config.go           # 配置结构体和加载
│   │   ├── crypto.go           # 配置加密
│   │   └── system_prompt.go    # 系统提示处理
│   │
│   ├── core/                   # 执行引擎
│   │   ├── execution.go        # LLMCore (重试/降级)
│   │   ├── errors.go           # 错误类型
│   │   └── results.go          # 结果类型
│   │
│   ├── openai/                 # OpenAI 客户端
│   │   └── client.go           # HTTP 客户端和流式处理
│   │
│   ├── plugin/                 # 插件系统
│   │   ├── plugin.go           # Plugin 接口
│   │   ├── manager.go          # PluginManager
│   │   ├── hooks.go            # HookRegistry
│   │   ├── approval.go         # 审批类型
│   │   ├── external.go         # 外部插件适配器
│   │   ├── loader.go           # 外部插件加载
│   │   ├── typed_hooks.go      # 类型化 Hook 包装
│   │   └── proto/              # RPC 协议
│   │       ├── types.go        # 类型定义
│   │       └── rpc.go          # RPC 实现
│   │
│   ├── republic/               # Facade API
│   │   └── republic.go         # 简化 API
│   │
│   ├── tape/                   # 审计日志
│   │   ├── manager.go          # TapeManager
│   │   ├── store.go            # TapeStore 接口
│   │   ├── entry.go            # TapeEntry 定义
│   │   ├── context.go          # TapeContext
│   │   ├── query.go            # TapeQuery
│   │   ├── file_store.go       # 文件存储
│   │   ├── sqlite_store.go     # SQLite 存储
│   │   ├── pg_store.go         # PostgreSQL 存储
│   │   └── ...
│   │
│   ├── tool/                   # 工具系统
│   │   ├── tool.go             # Tool 定义
│   │   ├── executor.go         # ToolExecutor
│   │   └── context.go          # ToolContext
│   │
│   ├── auth/                   # 认证
│   ├── credentials/            # 凭证管理
│   ├── cwd/                    # 工作目录管理
│   ├── logging/                # 日志
│   ├── policyinject/           # 策略注入
│   └── service/                # OS 服务管理
│
├── plugins/                    # 内置插件
│   ├── cli/                    # CLI 界面
│   │   ├── cli.go
│   │   ├── chat.go
│   │   ├── approver.go
│   │   └── renderer.go
│   │
│   ├── command/                # 命令拦截
│   │   └── command.go
│   │
│   ├── cron/                   # 定时任务
│   │   ├── cron.go
│   │   ├── executor.go
│   │   ├── job.go
│   │   ├── schedule.go
│   │   └── tools.go
│   │
│   ├── credentials/            # 凭证管理
│   │   └── credentials.go
│   │
│   ├── fs/                     # 文件系统
│   │   └── fs.go
│   │
│   ├── mcp/                    # MCP 桥接
│   │   └── mcp.go
│   │
│   ├── opapolicy/              # OPA 策略
│   │   ├── opapolicy.go
│   │   ├── engine_build.go
│   │   ├── reload.go
│   │   ├── cache.go
│   │   └── policies/           # 内嵌策略
│   │       └── default.rego
│   │
│   ├── shell/                  # Shell 执行
│   │   ├── shell.go
│   │   ├── shell_manager.go
│   │   └── env.go
│   │
│   ├── skill/                  # 技能系统
│   │   └── skill.go
│   │
│   ├── subagent/               # 子 Agent
│   │   └── subagent.go
│   │
│   ├── tape/                   # Tape 工具
│   │   └── tape.go
│   │
│   ├── toolsearch/             # 工具搜索
│   │   └── toolsearch.go
│   │
│   ├── webserver/              # Web 服务
│   │   ├── plugin.go
│   │   ├── server.go
│   │   ├── handlers.go
│   │   ├── approver_adapter.go
│   │   └── auth.go
│   │
│   ├── webtool/                # Web 工具
│   │   ├── web.go
│   │   ├── fetch.go
│   │   ├── render.go
│   │   └── search_ddg.go
│   │
│   ├── clawhub/                # ClawHub 技能市场
│   │   ├── clawhub.go
│   │   └── tools.go
│   │
│   ├── osinfo/                 # 系统信息
│   │   └── osinfo.go
│   │
│   └── powershell/             # PowerShell (Windows)
│       └── powershell.go
│
├── docs/                       # 文档
│   ├── tool-naming.md          # 工具命名规范
│   └── issues/                 # 设计文档
│       ├── 01-core-errors.md
│       ├── 03-tape-system.md
│       ├── 04-tool-system.md
│       ├── 06-core-execution.md
│       └── ...
│
├── examples/                   # 示例配置
│   └── permissive-policy.rego
│
├── bin/                        # 构建输出
├── config.example.toml         # 配置示例
├── AGENTS.md                   # 开发指南
└── README.md                   # 项目说明
```

## 关键文件说明

### 入口和命令

| 文件 | 说明 |
|------|------|
| `cmd/dmr/main.go` | 程序入口 |
| `cmd/dmr/root.go` | Cobra 根命令定义，全局标志 |
| `cmd/dmr/commands.go` | run, chat, plugins, version 命令 |
| `cmd/dmr/brain.go` | brain 命令实现 |
| `cmd/dmr/plugins.go` | buildPluginManager 函数 |
| `cmd/dmr/agent_builder.go` | buildAgent 函数 |

### 核心包

| 文件 | 说明 |
|------|------|
| `pkg/agent/loop.go` | Agent 主循环 (520+ 行核心代码) |
| `pkg/core/execution.go` | LLMCore 执行引擎 |
| `pkg/client/chat.go` | ChatClient 实现 |
| `pkg/tape/manager.go` | TapeManager 操作 |
| `pkg/tool/executor.go` | ToolExecutor 实现 |
| `pkg/plugin/manager.go` | PluginManager |
| `pkg/plugin/hooks.go` | HookRegistry |

### 插件

| 文件 | 说明 |
|------|------|
| `plugins/shell/shell.go` | shell 工具实现 (409 行) |
| `plugins/fs/fs.go` | fs 工具实现 |
| `plugins/opapolicy/opapolicy.go` | OPA 策略引擎 (455 行) |
| `plugins/skill/skill.go` | 技能系统 (278 行) |
| `plugins/webserver/plugin.go` | Web 服务插件 |
| `plugins/cli/cli.go` | CLI 插件 |

## 依赖关系

```
cmd/dmr
    ├── pkg/config
    ├── pkg/agent
    │       ├── pkg/client
    │       │       ├── pkg/core
    │       │       │       └── pkg/openai
    │       │       ├── pkg/tool
    │       │       └── pkg/tape
    │       ├── pkg/plugin
    │       └── pkg/tape
    └── pkg/plugin
            └── pkg/tool
```

## 构建

```bash
# 构建二进制
go build -o bin/dmr ./cmd/dmr/

# 运行
go run ./cmd/dmr/ run "Hello"

# 测试
go test ./...
```

## 配置文件结构

```toml
# 模型配置
[[models]]
name = "gpt4"
model = "gpt-4o"
api_key = "sk-..."
default = true

# Agent 配置
[agent]
max_steps = 20
max_token = 8000
system_prompt = "You are a helpful assistant"

# Tape 配置
[tape]
driver = "sqlite"
dsn = "~/.dmr/tapes.db"

# 插件配置
[[plugins]]
name = "shell"
enabled = true
[plugins.config]
timeout = 60
```
