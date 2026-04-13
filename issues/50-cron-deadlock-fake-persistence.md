# P1: Cron 插件死锁 + 虚假持久化 + 缺失重试

> `applyJobs()` 持有 `p.mu.Lock()` 时调用 `queueImmediateRun()` 再次 `Lock()`，Go Mutex 不可重入，确定性死锁。`BuiltinQueue` 声称"persistent queue using SQLite"但实际是纯内存 channel，序列化后丢弃数据。`Task.MaxAttempts`/`Attempts` 字段已定义但从未使用，retry 未实现。

## 问题

### 1. 确定性死锁

`plugins/cron/cron.go` 的调用链：

```
applyJobs()           → 持有 p.mu.Lock()
  └→ applyJobsIncremental()
       └→ queueImmediateRun()  → 尝试 p.mu.Lock() ← 死锁
```

具体代码路径：

```go
// plugins/cron/cron.go:441
func (p *Plugin) applyJobs() {
    p.mu.Lock()         // 获取锁
    defer p.mu.Unlock()
    // ...
    p.applyJobsIncremental(newJobs, removedJobs)
}

// plugins/cron/cron.go:579
func (p *Plugin) queueImmediateRun(job *Job) {
    p.mu.Lock()         // 再次获取锁 ← Go sync.Mutex 不可重入，死锁
    defer p.mu.Unlock()
    // ...
}
```

**触发条件**：任何带有 `run_on_load: true` 或 `@immediately` 调度的 job 在增量更新时被添加，都会触发此死锁。

### 2. BuiltinQueue 虚假持久化

`plugins/cron/queue.go:89-154`:

```go
// BuiltinQueue is a persistent queue backed by SQLite.  ← 注释声称 SQLite 持久化
type BuiltinQueue struct {
    ch chan *Task
    // 没有 SQLite 连接、没有数据库路径、没有持久化存储
}

func (q *BuiltinQueue) Enqueue(ctx context.Context, task *Task) error {
    data, _ := json.Marshal(task)
    _ = data  // ← 序列化后丢弃
    // In a full implementation, this would insert into SQLite.  ← 明确的 TODO

    select {
    case q.ch <- task:  // 放入内存 channel
        return nil
    default:
        return fmt.Errorf("queue full")
    }
}
```

**后果**：
- 进程重启后所有排队任务丢失
- `brain` 的 SIGHUP 热重载也会丢失任务（channel 被销毁）
- `queue full` 时任务被静默丢弃，没有持久化回退

### 3. Retry 机制只有骨架

`plugins/cron/queue.go:27-30`:

```go
type Task struct {
    MaxAttempts int  // 定义了最大重试次数
    Attempts    int  // 定义了当前尝试次数
    // ...
}
```

但 `Executor.executeTask()`（`plugins/cron/executor.go`）从未递增 `Attempts`，也从未检查 `MaxAttempts`。当 `runner.Run()` 失败时，任务被标记为 failed 并丢弃，不会重新入队。

### 4. 文件存储跨进程不安全

`plugins/cron/storage_file.go` 的 `UpsertJob()` 做了 load-modify-save 操作，受进程内 mutex 保护。但如果两个 DMR 进程（如用户手动运行 `dmr run` 和 `dmr brain` 后台服务）共享同一个配置文件，它们可以互相覆盖对方的 job 变更。

### 5. 过度复杂的同步原语

单个 cron 插件使用了 **8 个同步原语**：

```go
mu           sync.Mutex      // 主保护
startOnce    sync.Once       // 启动一次
asyncReloadWg sync.WaitGroup // 异步重载
locksMu      sync.Mutex      // job 锁的锁
executionsMu sync.Mutex      // 执行状态锁
jobLocks     map[string]*sync.Mutex  // per-job 锁
wg           sync.WaitGroup  // executor 等待组
stopCh       chan struct{}    // 停止信号
```

这种复杂度本身就是 bug 的温床（死锁就是证据）。

## 设计方案

### 1. 修复死锁

将需要锁的操作拆分为"收集阶段"和"执行阶段"：

```go
func (p *Plugin) applyJobs() {
    p.mu.Lock()
    // 阶段 1：收集需要立即运行的 jobs（仍在锁内）
    var immediateJobs []*Job
    for _, job := range newJobs {
        if job.RunOnLoad {
            immediateJobs = append(immediateJobs, job)
        }
    }
    p.mu.Unlock()

    // 阶段 2：触发立即运行（锁外）
    for _, job := range immediateJobs {
        p.queueImmediateRun(job)
    }
}
```

或者更简单——`queueImmediateRun` 改为 `queueImmediateRunLocked`，要求调用者已持锁，内部不再获取锁。

### 2. 实现真正的持久化队列

```go
type SQLiteQueue struct {
    db   *sql.DB
    ch   chan *Task    // 热路径仍用 channel
    path string
}

func (q *SQLiteQueue) Enqueue(ctx context.Context, task *Task) error {
    // 先写 SQLite（持久化）
    data, _ := json.Marshal(task)
    _, err := q.db.ExecContext(ctx,
        `INSERT INTO task_queue (id, data, status) VALUES (?, ?, 'pending')`,
        task.ID, data)
    if err != nil {
        return fmt.Errorf("persist task: %w", err)
    }

    // 再通知 channel（热路径）
    select {
    case q.ch <- task:
    default:
        // channel 满不是错误 — 下次 poll 会从 SQLite 捞出
    }
    return nil
}

// 启动时从 SQLite 恢复未完成的任务
func (q *SQLiteQueue) Recover(ctx context.Context) error {
    rows, _ := q.db.QueryContext(ctx,
        `SELECT data FROM task_queue WHERE status = 'pending'`)
    // ...恢复到 channel
}
```

### 3. 实现 retry

```go
func (e *Executor) executeTask(task *Task) {
    task.Attempts++
    err := e.runner.Run(ctx, task.TapeName, task.Prompt)
    if err != nil {
        if task.Attempts < task.MaxAttempts {
            // 指数退避重入队
            task.NextRun = time.Now().Add(backoff(task.Attempts))
            e.queue.Enqueue(ctx, task)
            return
        }
        slog.Error("task failed after max attempts", "task", task.ID, "attempts", task.Attempts)
    }
}
```

### 4. 简化同步原语

用 `chan struct{}` + `select` 替代部分 mutex，减少死锁表面积。考虑将 cron 调度循环改为 actor 模式（单 goroutine + command channel）。

## 实现步骤

- [ ] **紧急**：修复 `queueImmediateRun` 死锁（改为 locked 版本或释放锁后调用）
- [ ] 移除 `BuiltinQueue` 的误导性注释，或标记为 `// TODO: not yet persistent`
- [ ] 实现真正的 SQLite 持久化队列
- [ ] 实现 retry 逻辑（递增 Attempts，检查 MaxAttempts，指数退避）
- [ ] 为文件存储添加 flock 保护跨进程安全
- [ ] 重构同步原语，减少 mutex 嵌套层级
- [ ] 添加测试：死锁触发路径、队列持久化恢复、retry 逻辑

## 代价与风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| SQLite 队列增加 cron 插件对 SQLite 的依赖 | 确定 | 低 | DMR 已依赖 SQLite（tape store），非新增依赖 |
| Retry 导致失败任务反复执行 | 中 | 中 | MaxAttempts 默认 3；指数退避避免密集重试 |
| 简化同步原语可能引入新竞争 | 中 | 中 | 用 `go test -race` 覆盖，重点测试 SIGHUP 热重载路径 |

## 参考

- `plugins/cron/cron.go:441` — applyJobs（持锁）
- `plugins/cron/cron.go:579` — queueImmediateRun（再次获取锁）
- `plugins/cron/queue.go:89-154` — BuiltinQueue 虚假持久化
- `plugins/cron/executor.go` — executeTask 无 retry
- `plugins/cron/storage_file.go` — 文件存储无跨进程锁
- `docs/audit/03-architecture-design.md:178` — 审计文档已标记 cron 过于复杂
