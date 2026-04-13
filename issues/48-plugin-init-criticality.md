# P0: 插件初始化失败无关键性区分 — 安全插件静默降级

> `InitAll()` 对所有插件采用 warn-and-continue 模式，不区分关键插件（opapolicy、shell）与可选插件（osinfo）。OPA 策略引擎初始化失败时，agent 照常运行但没有安全策略——对以 "verify" 为核心理念的框架来说是致命的。外部插件加载在第一个失败时短路，后续插件永远不会被尝试。

## 问题

### 1. 所有调用点都是 warn-and-continue

全部 4 个调用点（`cmd/dmr/commands.go:30-31`、`cmd/dmr/commands.go:69-70`、`cmd/dmr/commands.go:130-131`、`cmd/dmr/brain.go:58-59`）均为：

```go
if err := pm.InitAll(context.Background(), pluginConfigs(app.cfg)); err != nil {
    slog.Warn("plugin InitAll failed", "error", err)
}
```

没有任何分支检查**哪个插件**失败了，也没有区分失败的严重程度。

### 2. InitAll 收集错误但不携带来源

`pkg/plugin/manager.go:42-54`:

```go
func (m *Manager) InitAll(ctx context.Context, configs map[string]map[string]any) error {
    var errs []error
    for _, p := range m.plugins {
        if err := p.Init(ctx, configs[p.Name()]); err != nil {
            errs = append(errs, fmt.Errorf("init %s: %w", p.Name(), err))
        }
    }
    return errors.Join(errs...)
}
```

返回的是 `errors.Join` 合并的错误，调用方无法按插件名判断哪些失败了。

### 3. 具体影响场景

| 失败的插件 | 后果 | 严重程度 |
|-----------|------|---------|
| `opapolicy` | 所有工具调用不经过策略检查，直接执行 | **P0 — 安全旁路** |
| `shell` | LLM 无法执行 shell 命令，但不会收到明确提示 | P1 |
| `cli` | 无审批者和聊天界面，agent 无法交互 | P1 |
| `tape` | 无 tape 查询工具，但 tape 存储本身仍工作 | P2 |
| `osinfo` | 系统信息不出现在 system prompt 中 | P3 — 可接受 |
| `toolsearch` | 扩展工具无法被发现 | P2 |

**opapolicy 静默失败**是最危险的场景：用户认为 OPA 策略在保护他们，但实际上策略引擎没有初始化，所有工具调用都在无策略状态下执行。

### 4. 外部插件加载短路

`pkg/plugin/loader.go:64`:

```go
func LoadExternalPlugins(mgr *Manager, configs []PluginConfig, ...) error {
    for _, cfg := range configs {
        // ... setup ...
        if err != nil {
            return fmt.Errorf("load external plugin %s: %w", cfg.Name, err)  // 第一个失败就返回
        }
    }
    return nil
}
```

如果配置了 3 个外部插件，第 1 个失败会导致第 2、3 个永远不被加载。这与 `InitAll` 的"尽力尝试所有"策略不一致。

## 设计方案

### 1. 插件分级

在 `PluginConfig` 中增加关键性标记：

```go
type PluginConfig struct {
    Name     string         `toml:"name"`
    Enabled  bool           `toml:"enabled"`
    Critical bool           `toml:"critical,omitempty"`  // 新增
    Path     string         `toml:"path,omitempty"`
    Config   map[string]any `toml:"config,omitempty"`
}
```

内置插件有默认关键性等级：

```go
var defaultCriticality = map[string]bool{
    "opa_policy": true,   // 安全核心 — 必须成功
    "cli":       true,   // 交互核心 — chat/run 模式必须成功
    "command":   true,   // 命令调度 — 必须成功
    "tape":      false,  // 降级可接受
    "toolsearch": false,
    "osinfo":    false,
}
```

### 2. InitAll 返回结构化结果

```go
type InitResult struct {
    Failed   []PluginError  // 所有失败的插件
    Critical []PluginError  // 其中标记为 critical 的
}

type PluginError struct {
    Name string
    Err  error
}

func (m *Manager) InitAll(ctx context.Context, configs map[string]map[string]any) (*InitResult, error) {
    result := &InitResult{}
    for _, p := range m.plugins {
        if err := p.Init(ctx, configs[p.Name()]); err != nil {
            pe := PluginError{Name: p.Name(), Err: err}
            result.Failed = append(result.Failed, pe)
            if m.isCritical(p.Name()) {
                result.Critical = append(result.Critical, pe)
            }
        }
    }
    if len(result.Critical) > 0 {
        return result, fmt.Errorf("critical plugins failed: %v", result.Critical)
    }
    return result, nil
}
```

### 3. 调用方区分处理

```go
result, err := pm.InitAll(ctx, pluginConfigs(app.cfg))
if err != nil {
    // 关键插件失败 — 终止启动
    slog.Error("critical plugin init failed", "errors", result.Critical)
    os.Exit(1)
}
if len(result.Failed) > 0 {
    // 非关键插件失败 — 警告但继续
    for _, f := range result.Failed {
        slog.Warn("optional plugin init failed", "plugin", f.Name, "error", f.Err)
    }
}
```

### 4. 修复外部插件加载短路

```go
func LoadExternalPlugins(mgr *Manager, configs []PluginConfig, ...) error {
    var errs []error
    for _, cfg := range configs {
        if err := loadOne(mgr, cfg, ...); err != nil {
            errs = append(errs, fmt.Errorf("external plugin %s: %w", cfg.Name, err))
            continue  // 继续尝试下一个
        }
    }
    return errors.Join(errs...)
}
```

## 实现步骤

- [ ] 在 `PluginConfig` 中增加 `Critical` 字段
- [ ] 定义内置插件的默认关键性映射
- [ ] 重构 `InitAll` 返回 `InitResult`
- [ ] 修改 4 个调用点，区分 critical 和 optional 失败
- [ ] 修复 `LoadExternalPlugins` 的短路问题（continue 替代 return）
- [ ] 在 `brain` 模式下，`cli` 不应标记为 critical（brain 不需要 CLI）
- [ ] 添加测试：模拟 critical 插件失败，验证进程退出

## 代价与风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| 原本能启动的配置因 critical 插件失败而拒绝启动 | 中 | 中 | 提供 `--force` 跳过 critical 检查；清晰的错误提示指导修复 |
| Critical 标记粒度难以统一 | 低 | 低 | 默认保守标记，用户可通过配置覆盖 |

## 参考

- `pkg/plugin/manager.go:42-54` — InitAll 实现
- `pkg/plugin/loader.go:64` — LoadExternalPlugins 短路
- `cmd/dmr/commands.go:30-31` — warn-and-continue 调用点
- `cmd/dmr/brain.go:58-59` — brain 模式调用点
- [#09 TapeStore.Append 错误](./09-tapestore-append-error.md) — 另一个静默失败问题
