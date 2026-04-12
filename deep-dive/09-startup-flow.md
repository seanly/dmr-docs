# 启动流程深入分析

## 概述

DMR 有三种启动模式：
1. **run 模式**: 单次执行
2. **chat 模式**: 交互式 REPL
3. **brain 模式**: 多通道后台服务

## 整体流程

```
main()
    │
    ▼
rootCmd.Execute()
    │
    ▼
PersistentPreRun: prepareConfigForCmd()
    │
    ├──► 加载配置文件
    ├──► 合并命令行参数
    └──► 验证配置
    │
    ▼
命令执行 (run/chat/brain)
    │
    ├──► 创建 PluginManager
    ├──► 注册内置插件
    ├──► 加载外部插件
    ├──► 初始化所有插件
    │
    ├──► 创建 TapeStore
    ├──► 创建 Agent
    │
    └──► 执行业务逻辑
         ├──► run: Agent.Run() 一次
         ├──► chat: 启动 REPL 循环
         └──► brain: 启动服务循环
    │
    ▼
清理
    ├──► ShutdownAll 插件
    └──► 关闭 Store
```

## 配置准备

```go
func prepareConfigForCmd(cmd *cobra.Command) {
    // 1. 加载配置
    if app.cfgFile != "" {
        app.cfg, _ = config.Load(app.cfgFile)
    } else {
        app.cfg = config.LoadDefault()
    }
    
    // 2. 合并命令行参数
    if app.flagModel != "" {
        // 更新默认模型
    }
    if app.flagAPIKey != "" {
        // 更新 API key
    }
    if app.flagAPIBase != "" {
        // 更新 API base
    }
    if app.flagSystem != "" {
        // 更新系统提示
    }
    if app.flagVerbose > 0 {
        app.cfg.Verbose = app.flagVerbose
    }
    
    // 3. 验证
    requireModelAndKey()
}

func requireModelAndKey() {
    dm := app.cfg.DefaultModel()
    if dm == nil || dm.Model == "" {
        log.Fatal("Error: Model required")
    }
    if dm.UsesClientCredentials() {
        if dm.APIBase == "" {
            log.Fatal("Error: api_base required for OAuth")
        }
        return
    }
    if dm.APIKey == "" {
        log.Fatal("Error: API key required")
    }
}
```

## 插件管理器构建

```go
func buildPluginManager(c *config.Config, hostService proto.HostService, skipPlugins ...string) *plugin.Manager {
    pm := plugin.NewManager()
    
    if app.flagNoPlugins {
        return pm
    }
    
    // 1. 内置插件工厂
    builtins := []builtinEntry{
        {"command", func() plugin.Plugin { return command.New() }},
        {"shell", func() plugin.Plugin { return execplugin.New() }},
        {"fs", func() plugin.Plugin { return fsplugin.New() }},
        {"tape", func() plugin.Plugin { return tapeplugin.New() }},
        {"subagent", func() plugin.Plugin { return subagent.New() }},
        {"skill", func() plugin.Plugin { return skill.New() }},
        {"cli", func() plugin.Plugin { return cliplugin.New() }},
        {"opa_policy", func() plugin.Plugin { return opapolicy.New() }},
        {"mcp", func() plugin.Plugin { return mcpplugin.New() }},
        {"credentials", func() plugin.Plugin { return credentialsplugin.New() }},
        {"webtool", func() plugin.Plugin { return webtoolplugin.New() }},
        {"webserver", func() plugin.Plugin { return webserverplugin.New() }},
        {"osinfo", func() plugin.Plugin { return osinfoplugin.New() }},
        {"powershell", func() plugin.Plugin { return powershellplugin.New() }},
        {"cron", func() plugin.Plugin { return cronplugin.New() }},
        {"clawhub", func() plugin.Plugin { return clawhubplugin.New() }},
        {"toolsearch", func() plugin.Plugin { return toolsearchplugin.New() }},
    }
    
    // 2. 构建启用集合
    enabledSet := make(map[string]bool)
    for _, pc := range c.Plugins {
        if pc.Enabled {
            enabledSet[pc.Name] = true
        }
    }
    
    // 3. 默认启用的核心插件
    defaultEnabled := []string{"command", "opa_policy", "cli", "tape", "toolsearch"}
    for _, name := range defaultEnabled {
        if _, exists := enabledSet[name]; !exists {
            enabledSet[name] = true
        }
    }
    
    // 4. 注册内置插件
    for _, b := range builtins {
        if !enabledSet[b.name] || skipSet[b.name] {
            continue
        }
        pm.Register(b.factory())
    }
    
    // 5. 加载外部插件
    var externalConfigs []plugin.ExternalPluginConfig
    for _, pc := range c.Plugins {
        if pc.Enabled && pc.Path != "" {
            externalConfigs = append(externalConfigs, plugin.ExternalPluginConfig{
                Name:    pc.Name,
                Path:    pc.Path,
                Enabled: pc.Enabled,
            })
        }
    }
    plugin.LoadExternalPlugins(pm, externalConfigs, hostService)
    
    return pm
}
```

## Agent 构建

```go
func buildAgent(c *config.Config, pm *plugin.Manager) (*agent.Agent, tape.TapeStore) {
    // 1. 创建 TapeStore
    storeConfig := tape.StoreConfig{
        Driver:         c.Tape.Driver,
        DSN:            c.Tape.DSN,
        Dir:            c.Tape.Dir,
        Workspace:      c.Workspace,
        EnableTSVector: c.Tape.EnableTSVector,
        TSVectorLang:   c.Tape.TSVectorLang,
        EnableFTS5:     c.Tape.EnableFTS5,
    }
    
    if storeConfig.Driver == "" {
        storeConfig.Driver = "sqlite"
        storeConfig.DSN = filepath.Join(home, ".dmr", "tapes.db")
    }
    
    store, err := tape.NewStore(storeConfig)
    if err != nil {
        store = tape.NewInMemoryTapeStore()
    }
    
    tape.SetTimezone(c.Tape.Timezone)
    
    return buildAgentWithStore(c, pm, store), store
}

func buildAgentWithStore(c *config.Config, pm *plugin.Manager, store tape.TapeStore) *agent.Agent {
    tm := tape.NewTapeManager(store)
    dm := c.DefaultModel()
    
    // 2. 创建 LLMCore
    llmCore := core.NewLLMCore(core.LLMCoreConfig{
        Model:        dm.Model,
        APIKey:       dm.APIKey,
        APIBase:      dm.APIBase,
        TokenURL:     dm.TokenURL,
        ClientID:     dm.ClientID,
        ClientSecret: dm.ClientSecret,
        Headers:      dm.Headers,
        MaxRetries:   3,
        Verbose:      c.Verbose,
    })
    
    // 3. 创建 ToolExecutor
    executor := tool.NewToolExecutor()
    executor.Verbose = c.Verbose
    
    // 4. 绑定策略钩子
    if pm.Hooks().HasHook(plugin.HookBeforeToolCall) {
        executor.BeforeToolCall = func(ctx context.Context, t *tool.Tool, args map[string]any, toolCtx *tool.ToolContext) error {
            return pm.Hooks().CallBeforeToolCall(ctx, plugin.BeforeToolCallArgs{...})
        }
    }
    
    if pm.Hooks().HasHook(plugin.HookBatchBeforeToolCall) {
        executor.BatchBeforeToolCall = func(ctx context.Context, items []tool.BatchCheckItem) map[int]error {
            result, _ := pm.Hooks().CallBatchBeforeToolCall(ctx, items)
            return result.(map[int]error)
        }
    }
    
    // 5. 创建 ChatClient
    chatClient := client.NewChatClient(llmCore, executor, tm)
    
    // 6. 构建系统提示
    defaultPrompt := config.DefaultSystemPrompt()
    userPrompt, _ := c.Agent.SystemPrompt.Resolve(c.ConfigDir)
    systemPromptBases, _ := c.Agent.ResolveSystemPrompts(c.ConfigDir)
    
    var finalBase string
    if userPrompt != "" {
        finalBase = defaultPrompt + "\n\n---\n\n" + userPrompt
    } else {
        finalBase = defaultPrompt
    }
    
    systemPrompt := plugin.ComposeSystemPrompt(context.Background(), pm.Hooks(), finalBase)
    
    // 7. 创建 Agent
    ag := agent.New(chatClient, tm, pm, agent.Config{
        MaxSteps:          c.Agent.MaxSteps,
        AgentPolicy:       c.Agent,
        SystemPrompt:      systemPrompt,
        SystemPromptBase:  finalBase,
        SystemPromptBases: systemPromptBases,
        TapeModels:        c.Agent.TapeModels,
        Workspace:         c.Workspace,
        Verbose:           c.Verbose,
        Models:            c.Models,
    })
    ag.SetExecutor(executor)
    
    return ag
}
```

## Run 模式流程

```go
func newRunCmd() *cobra.Command {
    return &cobra.Command{
        Use:   "run [prompt]",
        Run: func(cmd *cobra.Command, args []string) {
            prompt := strings.Join(args, " ")
            
            // 1. 构建插件管理器
            hostSvc := newHostServiceForExternalPlugins(app.cfg)
            pm := buildPluginManager(app.cfg, hostSvc, "webserver")
            pm.InitAll(context.Background(), pluginConfigs(app.cfg))
            defer pm.ShutdownAll(context.Background())
            
            // 2. 构建 Agent
            ag, store := buildAgent(app.cfg, pm)
            defer store.Close()
            
            // 3. 执行
            result, err := ag.Run(context.Background(), "default", prompt, 0)
            if err != nil {
                log.Fatalf("Error: %v", err)
            }
            
            fmt.Println(result.Output)
        },
    }
}
```

## Chat 模式流程

```go
func newChatCmd() *cobra.Command {
    return &cobra.Command{
        Use:   "chat",
        Run: func(cmd *cobra.Command, args []string) {
            // 1. 构建插件管理器
            hostSvc := newHostServiceForExternalPlugins(app.cfg)
            pm := buildPluginManager(app.cfg, hostSvc, "webserver", "subagent")
            pm.InitAll(context.Background(), pluginConfigs(app.cfg))
            defer pm.ShutdownAll(context.Background())
            
            // 2. 构建 Agent
            ag, store := buildAgent(app.cfg, pm)
            defer store.Close()
            
            // 3. 获取聊天提供者
            var chatProvider plugin.ChatProvider
            if pm.Hooks().HasHook("ProvideChatInterface") {
                result, _ := pm.Hooks().CallFirstResult(context.Background(), "ProvideChatInterface")
                chatProvider = result.(plugin.ChatProvider)
            }
            
            // 4. 创建并运行会话
            tape := "cli:" + sessionName
            session := chatProvider.CreateSession(ag, tape)
            session.Run(context.Background(), ag, tape)
        },
    }
}
```

## Brain 模式流程

```go
func newBrainCmd() *cobra.Command {
    return &cobra.Command{
        Use:   "brain",
        Run: func(cmd *cobra.Command, args []string) {
            for {
                restart, err := runBrainOnce()
                if err != nil {
                    log.Fatalf("Brain error: %v", err)
                }
                if !restart {
                    break
                }
                // 收到 SIGHUP，重新加载配置
                loadConfig()
            }
        },
    }
}

func runBrainOnce() (restart bool, err error) {
    // 1. 创建 HostService（延迟绑定）
    hostService := newHostServiceForExternalPlugins(app.cfg)
    
    // 2. 构建插件管理器（跳过 cli_approver）
    pm := buildPluginManagerWithVerbose(app.cfg, hostService, true, "cli_approver")
    pm.InitAll(context.Background(), pluginConfigs(app.cfg))
    wireHealthChecks(pm)
    
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
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, os.Interrupt, syscall.SIGTERM, syscall.SIGHUP)
    sig := <-sigCh
    
    if sig == syscall.SIGHUP {
        restart = true
    }
    
    // 7. 关闭
    shutCtx, cancel := context.WithTimeout(context.Background(), 8*time.Second)
    defer cancel()
    pm.ShutdownAll(shutCtx)
    pm.KillExternal()
    store.Close()
    
    return restart, nil
}
```

## 插件配置注入

```go
func pluginConfigs(c *config.Config) map[string]map[string]any {
    baseDir := strings.TrimSpace(c.ConfigDir)
    if baseDir == "" {
        baseDir = config.DefaultDir()
    }
    absBase, _ := filepath.Abs(baseDir)
    
    m := make(map[string]map[string]any)
    for _, pc := range c.Plugins {
        if !pc.Enabled {
            continue
        }
        merged := make(map[string]any)
        for k, v := range pc.Config {
            merged[k] = v
        }
        // 注入标准字段
        merged["config_base_dir"] = absBase
        merged["plugin_name"] = pc.Name
        merged["workspace"] = resolveWorkspace(c.Workspace, absBase)
        
        m[pc.Name] = merged
    }
    return m
}
```

## 关闭流程

```go
func (m *Manager) ShutdownAll(ctx context.Context) error {
    m.mu.RLock()
    defer m.mu.RUnlock()
    
    var errs []error
    // 逆序关闭
    for i := len(m.plugins) - 1; i >= 0; i-- {
        p := m.plugins[i]
        if err := p.Shutdown(ctx); err != nil {
            errs = append(errs, fmt.Errorf("shutdown plugin %q: %w", p.Name(), err))
        }
    }
    return errors.Join(errs...)
}

func (m *Manager) KillExternal() {
    m.mu.RLock()
    defer m.mu.RUnlock()
    
    for _, p := range m.plugins {
        if ep, ok := p.(*ExternalPlugin); ok {
            if ep.kill != nil {
                ep.kill()
            }
        }
    }
}
```
