# P1: State Map 服务定位器反模式 — 类型系统被架构性绕过

> `ToolContext.State` 作为 `map[string]any` 承载了 6 种运行时依赖，13+ 处消费者通过魔术字符串 + 类型断言获取服务。编译器无法捕获 key 拼写错误或类型变更，Go 的类型安全被实质性降级为动态语言水平。

## 问题

`ToolContext.State`（`pkg/tool/context.go`）是一个 `map[string]any`，被当作全局服务定位器使用。Agent 在循环启动时将自身和各种运行时依赖注入其中：

```go
// pkg/agent/loop.go:95-98
toolCtx.State["_runtime_workspace"] = a.config.Workspace
toolCtx.State["_tape_store"]        = a.tape.Store
toolCtx.State["_tape_manager"]      = a.tape
toolCtx.State["_runtime_agent"]     = a
```

### 全部消费者清单

| 魔术 Key | 消费者文件 | 断言目标类型 |
|----------|-----------|-------------|
| `_runtime_workspace` | `pkg/tool/context.go`, `pkg/tool/executor.go`, `plugins/shell/shell.go`, `plugins/fs/fs.go`, `plugins/powershell/powershell.go`, `plugins/credentials/credentials.go` | `string` |
| `_runtime_workspace` | `plugins/opapolicy/opapolicy.go` | 直接作为 `any` 传递，不断言 |
| `_runtime_agent` | `plugins/subagent/subagent.go` | `plugin.RuntimeAgent`（共享接口） |
| `_runtime_agent` | `plugins/toolsearch/toolsearch.go` | 本地 `toolDiscoveryAgent` 接口 |
| `_tape_store` | `plugins/tape/tape.go`（3 处） | `tape.TapeStore` |
| `_tape_manager` | `plugins/tape/tape.go` | `*tape.TapeManager`（具体指针类型） |
| `_runtime_env` | `plugins/credentials/credentials.go` | `map[string]string` 或 `map[string]any` |
| `_runtime_inject_tempfiles` | `plugins/credentials/credentials.go` | type switch |

### 核心风险

1. **编译时零保障**：Key 拼写错误（如 `_runtime_workpsace`）或类型变更（如 `TapeStore` 接口新增方法）不会产生任何编译错误，只在运行时静默失败
2. **断言不一致**：同一个 `_runtime_agent` 对象，`subagent` 用 `plugin.RuntimeAgent` 断言，`toolsearch` 用本地 `toolDiscoveryAgent` 接口断言。两个接口方法集不同，`Agent` 重构时编译器不会警告
3. **具体类型泄露**：`_tape_manager` 被断言为 `*tape.TapeManager`（具体指针），而非接口。如果 `TapeManager` 改为接口或换实现，所有消费者崩溃
4. **失败处理不一致**：部分消费者使用 `_, ok :=` 检查并 return nil（静默跳过），部分直接断言不检查（panic 风险）

### 与 #10（ToolContext 竞争条件）的关系

#10 聚焦于 State map 的并发读写竞争。本 issue 聚焦于 State map 作为服务定位器的设计缺陷——即使不存在并发问题，类型安全和可维护性问题同样严重。

## 设计方案

### 方案：ToolContext 增加结构化字段

将运行时依赖从 `map[string]any` 提升为 `ToolContext` 上的具名、有类型的字段：

```go
// pkg/tool/context.go

type ToolContext struct {
    Ctx        context.Context
    TapeName   string
    RunID      string
    Meta       map[string]string
    State      map[string]any     // 保留，仅用于插件自定义 state

    // 运行时依赖：从 State map 提升为具名字段
    Workspace   string              // 原 _runtime_workspace
    TapeStore   tape.TapeStore      // 原 _tape_store（接口类型）
    TapeControl TapeController      // 原 _tape_manager（接口，非具体指针）
    Agent       RuntimeServices     // 原 _runtime_agent（统一接口）

    // CWD 管理（已有）
    CwdManager    *cwd.Manager
    CwdPolicy     CwdPolicy
    CwdMustBeUnder string

    PluginContext  map[string]any
}
```

### 统一 Agent 接口

解决 `subagent` 和 `toolsearch` 使用不同接口的问题：

```go
// pkg/plugin/runtime.go（或 pkg/contract/runtime.go）

// RuntimeServices 合并了 RuntimeAgent 和 toolDiscoveryAgent 的能力
type RuntimeServices interface {
    // 来自 RuntimeAgent
    Run(ctx context.Context, tapeName, prompt string) (string, error)

    // 来自 toolDiscoveryAgent
    GetAllExtendedTools() []*tool.Tool
    DiscoverTool(tapeName, toolName string)
    IsToolDiscovered(tapeName, toolName string) bool
}
```

### TapeController 接口

将具体指针 `*tape.TapeManager` 替换为接口：

```go
// pkg/tape/controller.go

type TapeController interface {
    Compact(tapeName string, opts CompactOpts) error
    Handoff(tapeName string) error
    RecordChat(tapeName string, msgs []Message) error
}
```

## 迁移路径

由于消费者分布在 13+ 处，建议分阶段迁移：

### 阶段 1：增加字段，双写兼容

```go
// pkg/agent/loop.go — 同时写入新字段和旧 State
toolCtx.Workspace = a.config.Workspace
toolCtx.State["_runtime_workspace"] = a.config.Workspace  // 兼容

toolCtx.Agent = a
toolCtx.State["_runtime_agent"] = a  // 兼容
```

### 阶段 2：逐插件迁移消费者

将每个插件从 `ctx.State["_runtime_workspace"].(string)` 改为 `ctx.Workspace`，逐个 PR 迁移。

### 阶段 3：移除 State 中的 `_runtime_*` key

删除所有 `State["_runtime_*"]` 写入，`State` 仅保留给插件自定义用途。

## 实现步骤

- [ ] 定义 `RuntimeServices` 接口（合并 `RuntimeAgent` + `toolDiscoveryAgent`）
- [ ] 定义 `TapeController` 接口
- [ ] 在 `ToolContext` 上添加结构化字段
- [ ] Agent 循环双写新字段 + 旧 State
- [ ] 逐插件迁移：shell、fs、powershell、credentials、opapolicy、subagent、toolsearch、tape、command
- [ ] 移除 State 中的 `_runtime_*` key
- [ ] 移除各插件中的类型断言代码

## 代价与风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| 迁移期间新旧路径不一致 | 中 | 低 | 双写机制确保兼容；逐插件迁移减小变更范围 |
| `RuntimeServices` 接口过大 | 低 | 中 | 保持接口精简，仅包含实际被消费的方法 |
| 外部插件依赖 State key | 低 | 中 | 外部插件通过 RPC 通信，不直接访问 State |

## 参考

- `pkg/tool/context.go` — ToolContext 定义
- `pkg/agent/loop.go:95-98` — 当前注入点
- `pkg/plugin/runtime.go` — 现有 RuntimeAgent 接口
- `plugins/toolsearch/toolsearch.go:93-97` — toolDiscoveryAgent 本地接口
- [#10 ToolContext 并发竞争](./10-toolcontext-race-condition.md) — 并发维度的问题
