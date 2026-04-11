# P1: 插件接口重构 — 能力注册模式替代大接口（Plugin Interface Refactor）

> 解决 `DMRPluginInterface` 大接口反模式问题。插件被迫实现不需要的方法（如 vsphere 必须写空的 `RequestApproval`），新增能力需要所有插件更新。推荐采用能力注册模式（Capability Registration），插件自声明能力，DMR 按能力索引调用。

## 问题

当前 `DMRPluginInterface`（`pkg/plugin/proto/types.go`）是一个"God Interface"：

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

| 问题 | 影响 | 示例 |
|------|------|------|
| 大接口反模式 | 插件被迫实现不需要的方法 | vsphere 必须写空的 `RequestApproval` |
| 职责混乱 | 工具插件 vs 审批插件界限不清 | cli 需要审批，jenkins 不需要 |
| 扩展困难 | 新增能力需要修改接口 | 添加 Cron 能力需要所有插件更新 |
| 代码冗余 | 大量空实现 | 80% 的插件有 `return nil, "not implemented"` |

现状：需要审批的插件仅 cli、webserver、feishu（3 个），不需要审批的插件 N 个。每新增一个工具插件必须写 2 个空方法。

## DMR 的实现方案

### 方案对比

评估了三种方案：

| 维度 | 方案1 独立接口 | 方案2 接口组合 | **方案3 能力注册** |
|------|--------------|--------------|------------------|
| 接口数量 | 1 + N | 2^N - 1 | 1 + N |
| 空实现 | 无 | 无 | 无 |
| 扩展性 | 好 | 差（组合爆炸） | 好 |
| 向后兼容 | 好 | 差 | 好 |
| 类型安全 | 运行时 | 编译期 | 编译期 + 运行时 |

**推荐：方案3（能力注册模式）**

### 核心接口设计

```go
// pkg/plugin/proto/interface_v2.go

type Plugin interface {
    Name() string
    Version() string
    Init(ctx context.Context, config map[string]any) error
    Shutdown(ctx context.Context) error
    Capabilities() []Capability  // 自声明能力
}

type Capability string

const (
    CapTools    Capability = "tools"
    CapApproval Capability = "approval"
    CapChat     Capability = "chat"
    CapCron     Capability = "cron"
    CapWebhook  Capability = "webhook"
)
```

### 能力接口

```go
// pkg/plugin/proto/capabilities.go

type ToolProvider interface {
    ListTools(ctx context.Context) ([]Tool, error)
    ExecuteTool(ctx context.Context, toolName string, args map[string]any) (any, error)
}

type Approver interface {
    RequestApproval(ctx context.Context, req ApprovalRequest) (ApprovalResult, error)
    RequestBatchApproval(ctx context.Context, req BatchApprovalRequest) (BatchApprovalResult, error)
    SupportsInteractive() bool
}

type ChatProvider interface {
    Start(ctx context.Context, session ChatSession) error
    SendMessage(ctx context.Context, msg Message) error
    ReceiveMessage(ctx context.Context) (Message, error)
}

type CronScheduler interface {
    Schedule(ctx context.Context, job CronJob) (JobID, error)
    Unschedule(ctx context.Context, id JobID) error
    ListJobs(ctx context.Context) ([]CronJob, error)
}

type WebhookHandler interface {
    HandleWebhook(ctx context.Context, req WebhookRequest) (WebhookResponse, error)
    GetRoutes() []string
}
```

### ManagerV2（按能力索引）

```go
// pkg/plugin/manager_v2.go

type ManagerV2 struct {
    mu           sync.RWMutex
    plugins      map[string]proto.Plugin
    tools        map[string]proto.ToolProvider
    approvers    map[string]proto.Approver
    chats        map[string]proto.ChatProvider
    schedulers   map[string]proto.CronScheduler
    webhooks     map[string]proto.WebhookHandler
    capabilities map[string][]proto.Capability
}

func (m *ManagerV2) Register(plugin proto.Plugin) error {
    name := plugin.Name()
    m.plugins[name] = plugin

    for _, cap := range plugin.Capabilities() {
        switch cap {
        case proto.CapTools:
            if p, ok := plugin.(proto.ToolProvider); ok {
                m.tools[name] = p
            }
        case proto.CapApproval:
            if p, ok := plugin.(proto.Approver); ok {
                m.approvers[name] = p
            }
        // ... 其他能力同理
        }
    }
    return nil
}

// 安全调用 — 不会 panic
func (m *ManagerV2) ExecuteTool(ctx context.Context, pluginName, toolName string, args map[string]any) (any, error) {
    provider, ok := m.tools[pluginName]
    if !ok {
        return nil, fmt.Errorf("plugin %s not found or does not provide tools", pluginName)
    }
    return provider.ExecuteTool(ctx, toolName, args)
}
```

### LegacyAdapter（旧插件兼容层）

```go
// pkg/plugin/compat.go

type LegacyAdapter struct {
    name   string
    legacy proto.DMRPluginInterface
}

func (a *LegacyAdapter) Capabilities() []proto.Capability {
    return []proto.Capability{proto.CapTools, proto.CapApproval}
}

func (a *LegacyAdapter) ListTools(ctx context.Context) ([]proto.Tool, error) {
    req := &proto.ProvideToolsRequest{}
    resp := &proto.ProvideToolsResponse{}
    if err := a.legacy.ProvideTools(req, resp); err != nil {
        return nil, err
    }
    // 转换...
}

// 外部插件自动适配，无需修改代码
func loadExternalPlugin(path string) {
    legacy := loadPluginBinary(path)
    adapter := NewLegacyAdapter("myplugin", legacy)
    manager.Register(adapter)
}
```

### 插件实现示例

纯工具插件（vsphere 风格）— 不需要写 `RequestApproval` 空方法：

```go
type VSpherePlugin struct{ client *VSphereClient }

func (p *VSpherePlugin) Capabilities() []proto.Capability {
    return []proto.Capability{proto.CapTools}
}

func (p *VSpherePlugin) ListTools(ctx context.Context) ([]proto.Tool, error) { ... }
func (p *VSpherePlugin) ExecuteTool(ctx context.Context, name string, args map[string]any) (any, error) { ... }
```

多能力插件（feishu 风格）：

```go
type FeishuPlugin struct{ bots []*BotInstance }

func (p *FeishuPlugin) Capabilities() []proto.Capability {
    return []proto.Capability{proto.CapTools, proto.CapApproval, proto.CapChat}
}

// 实现 ToolProvider + Approver + ChatProvider
```

### 内部插件 vs 外部插件

| 插件类型 | 位置 | 当前接口 | 修改策略 |
|---------|------|---------|---------|
| 内部插件 | `plugins/*` | `plugin.Plugin` (Go 接口) | 同仓库统一修改 |
| 外部插件 | `dmr-plugin-*` (独立仓库) | `proto.DMRPluginInterface` (gRPC) | LegacyAdapter 适配，无需修改 |

需要修改的内部插件：
- `plugins/cli` — 实现 `Approver`, `ChatProvider`
- `plugins/webserver` — 实现 `Approver`, `ChatProvider`
- `plugins/opa_policy` — 保持现状（Policy 是核心功能，不是可选能力）

外部插件（vsphere、jenkins、ssh、tiddlywiki）全部通过 LegacyAdapter 自动适配，**无需修改**。

## 实现步骤

### Phase 1: 添加新接口（v2.1，向后兼容）

- [ ] 新增 `pkg/plugin/proto/interface_v2.go`（Plugin + Capability）
- [ ] 新增 `pkg/plugin/proto/capabilities.go`（ToolProvider/Approver/ChatProvider/CronScheduler/WebhookHandler）
- [ ] 新增 `pkg/plugin/manager_v2.go`（ManagerV2 + 能力索引）
- [ ] 新增 `pkg/plugin/compat.go`（LegacyAdapter）

### Phase 2: 内部插件迁移（v2.2）

- [ ] 修改 `plugins/cli` 实现新接口
- [ ] 修改 `plugins/webserver` 实现新接口
- [ ] 更新 `cmd/dmr/commands.go` 初始化代码
- [ ] 验证新接口稳定性

### Phase 3: 废弃旧接口（v2.3）

- [ ] 标记 `DMRPluginInterface` 为 Deprecated
- [ ] 文档推荐新接口 + 迁移指南
- [ ] 新插件全部使用新接口

### Phase 4: 移除旧接口（v3.0，长期）

- [ ] 完全切换到新接口
- [ ] 删除兼容层代码

## 解决的问题

- **单一职责**：插件只实现需要的能力，无空方法
- **自文档化**：`Capabilities()` 显式声明，一目了然
- **类型安全**：编译期接口检查 + 运行时安全调用
- **向后兼容**：LegacyAdapter 适配旧插件，无需修改
- **易于扩展**：新增能力只需添加接口和常量，不影响现有插件

## 代价与风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| 现有外部插件大量重复工作 | 低 | 高 | LegacyAdapter 自动适配，无需修改 |
| 内部插件修改引入 Bug | 中 | 中 | 单元测试覆盖 + 渐进式发布 |
| 运行时类型断言失败 | 低 | 低 | DMR 核心封装，Register 时统一检查 |
| 开发者学习成本 | 低 | 低 | 示例代码 + 迁移指南 |

## 参考

- DMR `pkg/plugin/proto/types.go` — 现有 DMRPluginInterface 定义
- DMR `pkg/plugin/manager.go` — 现有 PluginManager
- DMR `cmd/dmr/commands.go:166-264` — Plugin 注册
- DMR `plugins/cli/cli.go` — CLI 插件（Approver + Chat）
- DMR `plugins/webserver/` — WebServer 插件（Approver）
