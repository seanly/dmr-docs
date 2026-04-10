# P1: 隔离式工具执行 — 进程级沙箱

> 灵感来源：TapeAgents `RemoteEnvironment` 的进程级隔离 + UDS IPC

## TapeAgents 的做法

### RemoteEnvironment — 每个任务一个进程

```python
# tapeagents/remote_environment.py

class ProcessPoolManager:
    """管理 worker 进程池，每个任务一个独立进程"""
    
    def spawn_worker(self, worker_id, env_config, env_class):
        socket_path = os.path.join(self.socket_dir, f"worker_{worker_id}.sock")
        process = Process(
            target=ProcessPoolManager._task_worker,
            args=(env_config, env_class, socket_path, worker_id),
            daemon=True,  # 主进程退出时自动回收
        )
        process.start()

class EnvironmentServer:
    """FastAPI HTTP 服务器，接收外部请求，分发到 worker 进程"""
    # start_task_endpoint() → spawn worker → 等待 UDS socket 就绪 → 发送 start_task 命令
```

**Worker 进程通信协议**（`_task_worker()`）：
- 绑定 Unix Domain Socket（`socket.AF_UNIX`）
- 4 字节大端长度头 + pickle 序列化的请求/响应
- 支持命令：`step`、`actions`、`reset`、`start_task`、`shutdown`

**关键特性**：
1. **崩溃隔离**：工具进程 crash 不影响主 agent 进程
2. **资源限制**：可以对子进程施加 cgroup/rlimit 限制
3. **超时控制**：主进程可以 kill 超时的 worker 进程
4. **状态隔离**：每个任务的文件系统变更在独立进程中
5. **协议简单**：长度前缀 + pickle 序列化，低开销

## DMR 为什么特别需要这个

DMR 是 **DevOps Agent**，它执行的工具包括：
- `exec_command` — 执行任意 shell 命令
- `kubectl` — 操作 Kubernetes 集群
- `file_write` — 写入文件系统
- `deploy` — 触发部署

这些工具直接在 agent 主进程中执行。风险：

1. **工具挂起**：一个 `kubectl exec` 命令无响应 → 整个 agent 阻塞（参见 Issue #13）
2. **工具崩溃**：工具 panic → 如果 recover 不完整，agent 进程退出
3. **资源泄漏**：工具启动子进程后未清理 → 僵尸进程积累
4. **安全边界**：agent 和工具共享同一个进程空间、同一份凭证

## 修复方案

### 阶段一：工具执行器增加 sandbox 模式

```go
// pkg/tool/sandbox.go

type SandboxConfig struct {
    Enabled     bool          `toml:"enabled"`
    Mode        string        `toml:"mode"`      // "process" | "container" | "none"
    Timeout     time.Duration `toml:"timeout"`    // 单次工具执行超时
    MaxMemoryMB int           `toml:"max_memory"` // 内存限制
    WorkDir     string        `toml:"work_dir"`   // 工作目录隔离
}

type SandboxedExecutor struct {
    inner   ToolExecutor
    config  SandboxConfig
}

func (e *SandboxedExecutor) Execute(ctx context.Context, call ToolCall) (string, error) {
    switch e.config.Mode {
    case "process":
        return e.executeInProcess(ctx, call)
    case "container":
        return e.executeInContainer(ctx, call)
    default:
        return e.inner.Execute(ctx, call)
    }
}
```

### 阶段二：进程级隔离（Go 实现）

```go
// pkg/tool/process_sandbox.go

func (e *SandboxedExecutor) executeInProcess(ctx context.Context, call ToolCall) (string, error) {
    // 创建带超时的 context
    execCtx, cancel := context.WithTimeout(ctx, e.config.Timeout)
    defer cancel()
    
    // 通过子进程执行工具
    // 使用 Unix Domain Socket 进行 IPC
    socketPath := filepath.Join(os.TempDir(), fmt.Sprintf("dmr-tool-%s.sock", call.ID))
    defer os.Remove(socketPath)
    
    // 启动 worker 进程
    cmd := exec.CommandContext(execCtx, os.Args[0], "tool-worker",
        "--socket", socketPath,
        "--tool", call.Name,
        "--max-memory", fmt.Sprintf("%d", e.config.MaxMemoryMB),
    )
    cmd.SysProcAttr = &syscall.SysProcAttr{
        Setpgid: true,  // 新进程组，方便 kill
    }
    
    if err := cmd.Start(); err != nil {
        return "", fmt.Errorf("sandbox: failed to start worker: %w", err)
    }
    defer func() {
        // 确保清理：kill 进程组
        syscall.Kill(-cmd.Process.Pid, syscall.SIGKILL)
        cmd.Wait()
    }()
    
    // 等待 socket 就绪
    conn, err := waitForSocket(execCtx, socketPath, 5*time.Second)
    if err != nil {
        return "", fmt.Errorf("sandbox: worker failed to start: %w", err)
    }
    defer conn.Close()
    
    // 发送请求，接收结果
    result, err := sendToolRequest(conn, call)
    if err != nil {
        return "", fmt.Errorf("sandbox: tool execution failed: %w", err)
    }
    
    return result, nil
}
```

### 阶段三：容器级隔离（可选，高安全场景）

```go
// pkg/tool/container_sandbox.go

func (e *SandboxedExecutor) executeInContainer(ctx context.Context, call ToolCall) (string, error) {
    // 使用 Docker/Podman 运行工具
    containerName := fmt.Sprintf("dmr-tool-%s-%s", call.Name, call.ID)
    
    args := []string{
        "run", "--rm",
        "--name", containerName,
        "--memory", fmt.Sprintf("%dm", e.config.MaxMemoryMB),
        "--cpus", "1",
        "--network", "none", // 默认无网络（可按工具配置开放）
        "--read-only",       // 只读文件系统
        "-v", fmt.Sprintf("%s:/workspace", e.config.WorkDir),
        "dmr-tool-runner:latest",
        "--tool", call.Name,
        "--args", encodeArgs(call.Arguments),
    }
    
    cmd := exec.CommandContext(ctx, "docker", args...)
    output, err := cmd.CombinedOutput()
    return string(output), err
}
```

### 工具配置：按工具设置隔离级别

```toml
# config.toml

[tools.exec_command]
sandbox = "process"        # 进程隔离
timeout = "30s"
max_memory_mb = 256

[tools.kubectl]
sandbox = "process"        # 进程隔离
timeout = "60s"
max_memory_mb = 512
network = true             # 需要网络访问

[tools.search]
sandbox = "none"           # 无需隔离（只读操作）

[tools.untrusted_plugin]
sandbox = "container"      # 最高级别隔离
timeout = "120s"
max_memory_mb = 1024
network = false
```

## 与 Issue #13 (Tool Timeout) 的关系

Issue #13 提出了工具超时的问题。本方案是 #13 的**超集**：
- #13 只解决超时 → 本方案增加了崩溃隔离、资源限制、安全边界
- 两者可以合并实现：`SandboxedExecutor` 内置 timeout 支持

## 为什么值得做

| 维度 | 说明 |
|------|------|
| **解决什么问题** | 工具执行与 agent 共享进程空间，工具异常可瘫痪整个 agent |
| **安全价值** | 限制工具的资源、网络、文件系统访问——对 DevOps agent 至关重要 |
| **阶段一成本** | 中（需要新的 executor wrapper，但不改现有工具接口） |
| **阶段二成本** | 高（进程 IPC 复杂度） |
| **阶段三成本** | 中（Docker API 成熟，但引入容器依赖） |
| **DMR 特有价值** | DMR 运行在生产环境中，工具可能操作线上资源；隔离是安全红线 |
| **可扩展性** | none → process → container 三级，按需升级 |

## 参考

- TapeAgents `RemoteEnvironment`：`tapeagents/remote_environment.py`
- TapeAgents Worker 进程：`tapeagents/remote_environment.py`
- DMR 工具执行：`pkg/tool/executor.go`
- DMR Issue #13（工具超时）：`./13-tool-timeout-cancellation.md`
