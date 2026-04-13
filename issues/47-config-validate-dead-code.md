# P1: Config.Validate() 死代码 — 配置验证从未执行

> `Config.Validate()` 实现了重复 model 名检测、无效 tape driver 检查、负数 max_steps 拦截、时区验证，但全代码库无任何调用点。`LoadDefault()` 在解析失败时静默回退到默认配置。违反 fail-fast 原则，错误配置在运行时才暴露为难以诊断的行为。

## 问题

### 1. Validate() 从未被调用

`pkg/config/config.go:340-386` 定义了 `Config.Validate()` 方法，包含以下检查：

- 重复 model 名检测
- 无效 tape driver 字符串
- `max_steps` 为负数
- 无效时区字符串

但在全代码库中搜索 `.Validate()`、`cfg.Validate`、`config.Validate`，**零命中**。`Load()` 和 `LoadDefault()` 都不调用它。这是纯粹的死代码。

### 2. LoadDefault() 静默吞掉解析错误

```go
// pkg/config/config.go:498
func LoadDefault() (*Config, error) {
    cfg, err := Load(path)
    if err != nil {
        cfg = DefaultConfig()  // 解析失败？静默回退到默认值
    }
    // ...
}
```

用户的 TOML 文件有语法错误（如缺少引号、括号不匹配）时，不会收到任何错误提示——DMR 会默默使用默认配置启动。这导致：

- 用户以为自己的配置生效了，但实际上在用默认值
- 调试时极其困惑：`dmr config show` 可能显示默认值，用户不知道自己的配置被跳过了

### 3. 插件配置也无验证

`PluginConfig.Config` 是 `map[string]any`：

```go
type PluginConfig struct {
    Name    string         `toml:"name"`
    Enabled bool           `toml:"enabled"`
    Path    string         `toml:"path,omitempty"`
    Config  map[string]any `toml:"config,omitempty"`  // 无 schema
}
```

- 无 required 字段检查：缺少必要配置项时插件用零值运行
- 无类型检查：`include_shell = "yes"`（string 而非 bool）的断言静默失败，功能被禁用
- 无 unknown key 检测：`inclued_shell`（拼写错误）被静默忽略
- `BindConfig[T]` 通过 JSON round-trip 转换，静默丢弃未知字段和使用零值默认

### 4. 具体影响场景

| 配置错误 | 预期行为 | 实际行为 |
|----------|---------|---------|
| 重复 model 名 | 报错 | 静默使用第一个匹配 |
| `tape.driver = "postgre"` (拼写错误) | 报错 | 运行时 `buildAgent` 失败，错误信息不指向配置 |
| `agent.max_steps = -1` | 报错 | agent 循环行为未定义 |
| `agent.timezone = "Asia/Toky0"` | 报错 | tape 时间格式异常 |
| TOML 语法错误 | 报错并退出 | 静默使用默认配置 |
| `plugins.config.inclued_shell = true` | 提示未知 key | 静默忽略 |

## 设计方案

### 1. 在 Load/LoadDefault 中调用 Validate

```go
// pkg/config/config.go

func Load(path string) (*Config, error) {
    cfg := DefaultConfig()
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("read config: %w", err)
    }
    if err := toml.Unmarshal(data, cfg); err != nil {
        return nil, fmt.Errorf("parse config %s: %w", path, err)  // 不再静默回退
    }
    if err := cfg.Validate(); err != nil {
        return nil, fmt.Errorf("validate config: %w", err)
    }
    return cfg, nil
}
```

### 2. LoadDefault 区分"文件不存在"和"文件损坏"

```go
func LoadDefault() (*Config, error) {
    cfg, err := Load(path)
    if errors.Is(err, os.ErrNotExist) {
        return DefaultConfig(), nil  // 文件不存在：用默认值（合理）
    }
    if err != nil {
        return nil, err  // 文件存在但无法解析：报错（不再静默回退）
    }
    return cfg, nil
}
```

### 3. 增强 Validate 覆盖范围

```go
func (c *Config) Validate() error {
    var errs []error

    // 现有检查（已实现但未调用）
    errs = append(errs, c.validateModels()...)
    errs = append(errs, c.validateTape()...)
    errs = append(errs, c.validateAgent()...)

    // 新增：插件配置基础验证
    for _, pc := range c.Plugins {
        if pc.Name == "" {
            errs = append(errs, fmt.Errorf("plugin entry missing name"))
        }
        if pc.Path != "" && !pc.Enabled {
            // 配置了外部插件路径但 enabled=false，可能是误操作
            slog.Warn("external plugin configured but disabled",
                "name", pc.Name, "path", pc.Path)
        }
    }

    return errors.Join(errs...)
}
```

## 实现步骤

- [ ] 在 `Load()` 末尾调用 `cfg.Validate()`
- [ ] 修改 `LoadDefault()` 区分文件不存在 vs 解析错误
- [ ] 在 CLI 调用方（`cmd/dmr/root.go`）正确处理配置错误（打印清晰提示并退出）
- [ ] 增加 `dmr config check` 子命令，独立验证配置文件
- [ ] 为 Validate() 添加单元测试（当前无测试）
- [ ] 考虑为 TOML strict mode 启用 `DisallowUnknownFields`

## 代价与风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| 原本正常启动的配置在新版本报错 | 中 | 中 | 首版本将新增验证改为 warn 而非 error；下版本升级为 error |
| `DisallowUnknownFields` 破坏带注释的配置 | 低 | 中 | TOML 注释是 `#` 不受影响；仅影响真正的未知 key |

## 参考

- `pkg/config/config.go:340-386` — Validate() 实现（死代码）
- `pkg/config/config.go:405` — Load() 不调用 Validate
- `pkg/config/config.go:491-498` — LoadDefault() 静默回退
- `pkg/plugin/plugin.go:30-42` — BindConfig 静默丢弃未知字段
