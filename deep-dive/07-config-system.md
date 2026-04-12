# 配置系统深入分析

## 概述

DMR 使用 TOML 格式的配置文件，支持多模型、插件配置和高级选项。

## 配置结构

```toml
# 旧版单模型配置（向后兼容）
model = "gpt-4o"
api_key = "sk-..."
api_base = "https://api.openai.com/v1"

# 新版多模型列表
[[models]]
name = "gpt4"
model = "gpt-4o"
api_key = "sk-..."
api_base = "https://api.openai.com/v1"
default = true
max_token = 8000
handoff_threshold = 0.8

[[models]]
name = "claude"
model = "claude-3-sonnet"
api_key = "sk-..."
api_base = "https://api.anthropic.com/v1"

# 工作空间
workspace = "~/projects"

# Agent 配置
[agent]
max_steps = 20
max_token = 8000
handoff_threshold = 0.8
system_prompt = "You are a helpful assistant"

# Tape 配置
[tape]
driver = "sqlite"  # mem, file, sqlite, pg, mysql
dsn = "~/.dmr/tapes.db"
timezone = "Asia/Shanghai"
enable_fts5 = true

# 插件配置
[[plugins]]
name = "shell"
enabled = true
[plugins.config]
interactive = true
timeout = 60

[[plugins]]
name = "fs"
enabled = true

[[plugins]]
name = "webtool"
enabled = true
[plugins.config]
proxy = "http://127.0.0.1:1087"
```

## Config 结构体

```go
type Config struct {
    // 旧版字段
    Model   string
    APIKey  string
    APIBase string
    
    // 新版多模型
    Models    []ModelConfig
    Workspace string
    Agent     AgentConfig
    Tape      TapeConfig
    Plugins   []PluginConfig
    Verbose   int
    
    // 运行时字段（不序列化）
    ConfigDir string
}
```

## 模型配置

```go
type ModelConfig struct {
    Name                string
    Model               string
    APIKey              string
    APIBase             string
    Default             bool
    MaxToken            int     // 上下文预算
    HandoffThreshold    float64 // 自动 handoff 阈值
    CompletionMaxTokens int     // 完成 token 限制
    ToolResultMaxChars  int     // 工具结果截断
    
    // OAuth2 客户端凭证
    TokenURL     string
    ClientID     string
    ClientSecret string
    
    // 额外 HTTP 头
    Headers map[string]string
}
```

### OAuth2 支持

```go
func (m *ModelConfig) UsesClientCredentials() bool {
    return m.TokenURL != "" && m.ClientID != "" && m.ClientSecret != ""
}

func (m *ModelConfig) OAuthClientCredentialsIncomplete() bool {
    n := 0
    if m.TokenURL != "" { n++ }
    if m.ClientID != "" { n++ }
    if m.ClientSecret != "" { n++ }
    return n > 0 && n < 3
}
```

### 配置解析优先级

```go
func (m *ModelConfig) ResolveContextLimit(agentCfg AgentConfig) int {
    if m.MaxToken > 0 {
        return m.MaxToken
    }
    return agentCfg.MaxToken
}

func (m *ModelConfig) ResolveHandoffThreshold(agentCfg AgentConfig) float64 {
    if m.HandoffThreshold > 0 {
        return m.HandoffThreshold
    }
    if agentCfg.HandoffThreshold > 0 {
        return agentCfg.HandoffThreshold
    }
    return 0.8
}
```

## Agent 配置

```go
type AgentConfig struct {
    MaxSteps            int
    MaxToken            int
    HandoffThreshold    float64
    CompletionMaxTokens int
    ToolResultMaxChars  int
    
    // 系统提示（支持字符串或文件列表）
    SystemPromptRaw interface{}
    SystemPrompt    SystemPromptValue
    
    // 每 tape 系统提示
    SystemPromptsRaw map[string]interface{}
    SystemPrompts    map[string]SystemPromptValue
    
    // 每 tape 模型选择
    TapeModels map[string]string
}
```

### 系统提示值

```go
type SystemPromptValue struct {
    Raw   string   // 直接字符串值
    Files []string // 文件路径列表
}

func (s *SystemPromptValue) Resolve(baseDir string) (string, error) {
    if len(s.Files) == 0 {
        return s.Raw, nil
    }
    
    var parts []string
    for _, f := range s.Files {
        path := f
        if !filepath.IsAbs(path) {
            path = filepath.Join(baseDir, path)
        }
        data, _ := os.ReadFile(path)
        parts = append(parts, string(data))
    }
    return strings.Join(parts, "\n\n"), nil
}
```

## Tape 配置

```go
type TapeConfig struct {
    Driver         string   // mem, file, sqlite, pg, mysql
    DSN            string   // 数据库连接字符串
    Dir            string   // 文件驱动目录
    EnableTSVector bool     // PostgreSQL 全文搜索
    TSVectorLang   string   // tsvector 语言配置
    Timezone       string   // IANA 时区名
    EnableFTS5     FTS5Mode // SQLite FTS5: true/false/auto
}
```

### FTS5Mode

```go
type FTS5Mode struct {
    value string // "true", "false", "auto"
}

func NewFTS5Mode(s string) FTS5Mode {
    switch strings.ToLower(s) {
    case "true", "1", "yes", "on":
        return FTS5True
    case "auto", "automatic":
        return FTS5Auto
    default:
        return FTS5False
    }
}
```

## 插件配置

```go
type PluginConfig struct {
    Name    string
    Enabled bool
    Path    string         // 外部插件路径
    Config  map[string]any // 插件特定配置
}
```

### 默认配置

```go
func DefaultConfig() *Config {
    return &Config{
        Agent: AgentConfig{MaxSteps: 20},
        Tape: TapeConfig{
            EnableFTS5: FTS5True,
        },
        Plugins: []PluginConfig{
            {Name: "command", Enabled: true},
            {Name: "opa_policy", Enabled: true, Config: map[string]any{
                "policies":       []string{"etc/policies"},
                "approvals_file": "var/lib/opapolicy/approvals.json",
            }},
            {Name: "cli", Enabled: true},
            {Name: "tape", Enabled: true},
            {Name: "toolsearch", Enabled: true},
            {Name: "shell", Enabled: false},
            {Name: "fs", Enabled: false},
            {Name: "subagent", Enabled: false},
            {Name: "skill", Enabled: false, Config: map[string]any{
                "paths": []string{"skills/local"},
            }},
            // ...
        },
    }
}
```

## 配置加载

```go
func Load(path string) (*Config, error) {
    data, _ := os.ReadFile(path)
    
    var cfg Config
    if err := toml.Unmarshal(data, &cfg); err != nil {
        return nil, err
    }
    
    cfg.ConfigDir = filepath.Dir(path)
    
    // 向后兼容：单模型 → 多模型
    if len(cfg.Models) == 0 && cfg.Model != "" {
        cfg.Models = []ModelConfig{{
            Name:    cfg.Model,
            Model:   cfg.Model,
            APIKey:  cfg.APIKey,
            APIBase: cfg.APIBase,
            Default: true,
        }}
    }
    
    // 默认工作空间
    if cfg.Workspace == "" {
        cfg.Workspace = filepath.Join(cfg.ConfigDir, "workspace")
    }
    
    // 解析 SystemPromptRaw
    cfg.Agent.SystemPrompt = parseSystemPromptValue(cfg.Agent.SystemPromptRaw)
    
    // 解析 SystemPromptsRaw
    cfg.Agent.SystemPrompts = parseSystemPromptsMap(cfg.Agent.SystemPromptsRaw)
    
    return &cfg, nil
}
```

## 配置验证

```go
func (c *Config) Validate() error {
    // 检查重复默认模型
    defaultCount := 0
    namesSeen := make(map[string]bool)
    for _, m := range c.Models {
        if namesSeen[m.Name] {
            return fmt.Errorf("duplicate model name: %q", m.Name)
        }
        if m.Default {
            defaultCount++
        }
        if m.OAuthClientCredentialsIncomplete() {
            return fmt.Errorf("model %q: incomplete OAuth credentials", m.Name)
        }
    }
    if defaultCount > 1 {
        return fmt.Errorf("multiple default models")
    }
    
    // Agent 配置
    if c.Agent.MaxSteps < 0 {
        return fmt.Errorf("max_steps must be non-negative")
    }
    
    // Tape driver
    switch c.Tape.Driver {
    case "", "mem", "memory", "file", "sqlite", "pg", "postgres", "mysql":
        // valid
    default:
        return fmt.Errorf("unknown tape driver: %q", c.Tape.Driver)
    }
    
    // 时区
    if c.Tape.Timezone != "" {
        if _, err := time.LoadLocation(c.Tape.Timezone); err != nil {
            return fmt.Errorf("invalid timezone: %w", err)
        }
    }
    
    return nil
}
```

## 配置路径

```go
func DefaultDir() string {
    home, _ := os.UserHomeDir()
    return filepath.Join(home, ".dmr")
}

func DefaultPath() string {
    return filepath.Join(DefaultDir(), "config.toml")
}

func ExistingDefaultConfigPath() (string, bool) {
    p := filepath.Join(DefaultDir(), "config.toml")
    if _, err := os.Stat(p); err == nil {
        return p, true
    }
    return "", false
}
```

## 配置初始化

```go
func InitDefault() (string, error) {
    path := DefaultPath()
    dir := DefaultDir()
    
    os.MkdirAll(dir, 0755)
    
    // 初始化 V2 目录结构
    initV2Directories(dir)
    
    cfg := DefaultConfig()
    data, _ := toml.Marshal(cfg)
    
    header := "# DMR configuration (TOML)\n"
    os.WriteFile(path, []byte(header+string(data)), 0644)
    
    return path, nil
}

func initV2Directories(baseDir string) error {
    dirs := []string{
        "credentials",
        "cron",
        "var/log",
        "var/spool/history",
        "etc/policies",
        "tape",
        "workspace",
    }
    for _, d := range dirs {
        os.MkdirAll(filepath.Join(baseDir, d), 0755)
    }
    return nil
}
```

## 机密密封

```go
func EnsureWorkspace(cfg *Config, configPath string) error {
    if cfg.Workspace != "" {
        os.MkdirAll(cfg.Workspace, 0755)
    }
    
    // 密封明文秘钥
    if sealed, err := SealSecrets(cfg, configPath); err != nil {
        return err
    } else if sealed > 0 {
        fmt.Printf("Sealed %d secret(s) in %s\n", sealed, configPath)
    }
    
    return nil
}
```
