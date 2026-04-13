# P2: Args Map 命名空间污染 — Credential 注入混入工具参数

> `credentials` 插件通过 `BeforeToolCall` hook 向工具的 `args map[string]any` 注入 `_runtime_env` 和 `_runtime_inject_tempfiles`。LLM 传来的业务参数与运行时元数据混在同一个 map 中，形成隐式耦合。消费者（shell、fs）必须"知道"这些 magic key 的存在，verbose logging 被迫添加 redaction 逻辑。

## 问题

### 1. 注入路径

`plugins/credentials/credentials.go` 在 `BeforeToolCall` hook（priority 50，低于 OPA 的 100）中解析 `credential_bindings` 参数，将凭据内容注入回 args：

```go
// plugins/credentials/credentials.go:131-173

// 注入环境变量
args["_runtime_env"] = envMap        // map[string]string

// 注入临时文件（SSH key、secret file）
args["_runtime_inject_tempfiles"] = tempFiles  // 追加到已有列表
```

### 2. 消费者隐式耦合

Shell 和 FS 等工具必须知道检查这些 magic key：

```go
// plugins/shell/shell.go — 执行命令前检查注入的环境变量
if envRaw, ok := args["_runtime_env"]; ok {
    env := envRaw.(map[string]string)
    // 将 env 注入到 exec.Cmd.Env 中
}

// 同样检查 _runtime_inject_tempfiles
```

这创造了一个**隐式协议**：credential 插件写入 magic key → 工具插件读取 magic key，但这个协议没有被任何接口或类型定义——完全靠约定。

### 3. verbose logging 被迫做 redaction

`pkg/tool/executor.go` 在日志中显示工具参数时，必须专门排除这些 key：

```go
// pkg/tool/executor.go — 日志 redaction
// 需要排除 _runtime_env 和 _runtime_inject_tempfiles 避免泄露凭据
```

如果新增了类似的 magic key 但忘记在 executor 中加 redaction，就会在日志中泄露凭据。

### 4. OPA 策略看到污染后的 args

`BeforeToolCall` 的优先级顺序：OPA(100) > Credentials(50)。OPA 在 credential 注入**之前**评估，所以看到的是干净的 args。这个顺序是正确的，但如果优先级被不小心调整，OPA 会看到注入后的 args，导致策略匹配出错。这种脆弱性源于 args map 承担了双重职责。

### 5. 与 LLM 参数冲突的可能

虽然 `_runtime_` 前缀降低了冲突概率，但 `args map[string]any` 的内容来源是 LLM 的 JSON 输出。如果 LLM 输出中包含 `_runtime_env` key（恶意 prompt injection 或意外），它会被 credential 插件覆盖或被工具误读。

## 设计方案

### 方案：使用 ToolContext 传递运行时注入

将运行时注入的数据从 `args` 移到 `ToolContext`：

```go
// pkg/tool/context.go

type ToolContext struct {
    // ...existing fields...

    // 运行时注入（由 credential 插件填充）
    RuntimeEnv       map[string]string  // 环境变量注入
    RuntimeTempFiles []TempFileSpec     // 临时文件注入
}

type TempFileSpec struct {
    EnvKey  string // 指向此文件路径的环境变量名
    Content []byte // 文件内容（凭据）
    Mode    os.FileMode
}
```

### Credential 插件修改

```go
// plugins/credentials/credentials.go

func (p *Plugin) beforeToolCall(ctx context.Context, args BeforeToolCallArgs) error {
    toolCtx := args.ToolCtx  // 改用 ToolContext 而非 args map

    // 解析 credential_bindings
    bindings := args.Args["credential_bindings"]
    // ...resolve credentials...

    // 注入到 ToolContext 而非 args
    toolCtx.RuntimeEnv = envMap
    toolCtx.RuntimeTempFiles = append(toolCtx.RuntimeTempFiles, tempFiles...)

    // 从 args 中移除 credential_bindings（已处理完毕）
    delete(args.Args, "credential_bindings")

    return nil
}
```

### 工具插件修改

```go
// plugins/shell/shell.go

func (p *ShellPlugin) shellHandler(ctx *tool.ToolContext, args map[string]any) (any, error) {
    cmd := exec.Command(...)

    // 从 ToolContext 获取，而非 args
    if len(ctx.RuntimeEnv) > 0 {
        cmd.Env = append(os.Environ(), mapToEnvSlice(ctx.RuntimeEnv)...)
    }

    // 临时文件处理
    for _, tf := range ctx.RuntimeTempFiles {
        path := writeTempFile(tf)
        cmd.Env = append(cmd.Env, tf.EnvKey+"="+path)
        defer os.Remove(path)
    }
}
```

### 日志自动干净

由于运行时数据不再在 args 中，`executor.go` 的日志自动变干净，不需要 redaction 逻辑。

## 实现步骤

- [ ] 在 `ToolContext` 上添加 `RuntimeEnv` 和 `RuntimeTempFiles` 字段
- [ ] 修改 `credentials` 插件的 `beforeToolCall`，写入 ToolContext 而非 args
- [ ] 修改 `shell` 插件读取 ToolContext 而非 args
- [ ] 修改 `fs` 插件读取 ToolContext 而非 args
- [ ] 修改 `powershell` 插件读取 ToolContext 而非 args
- [ ] 移除 `executor.go` 中的 `_runtime_*` redaction 逻辑
- [ ] 验证 OPA 策略评估不受影响（args 不再包含 runtime key）
- [ ] 确保 `BeforeToolCallArgs.ToolCtx` 类型化（联动 #46）

## 代价与风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| 外部插件通过 RPC 的 CallTool 不传 ToolContext | 中 | 中 | 外部插件的 credential 通过 HostService.GetCredentialSecret 获取，不经过此路径 |
| 遗漏某个工具的 args 读取未迁移 | 低 | 低 | grep `_runtime_env` 确保全部清理 |

## 参考

- `plugins/credentials/credentials.go:131-173` — 当前注入逻辑
- `plugins/shell/shell.go:185` — Shell 读取 _runtime_env
- `plugins/fs/fs.go:191` — FS 读取 _runtime_workspace
- `pkg/tool/executor.go` — redaction 逻辑
- [#45 State Map 服务定位器](./45-state-service-locator.md) — ToolContext 结构化的整体方案
