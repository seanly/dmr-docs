# P0: TapeStore.Append 不返回错误 — 静默数据丢失

> 灵感来源：TapeAgents `execute_agent()` 的 tape 一致性保障 + 不可变 append 语义

## 问题本质

这是 DMR 最严重的架构级缺陷。

`TapeStore` 接口定义（`pkg/tape/store.go:15`）：

```go
type TapeStore interface {
    ListTapes() []string
    Reset(tape string)
    FetchAll(tape string, opts *FetchOpts) ([]TapeEntry, error)
    Append(tape string, entry TapeEntry)  // ← 没有 error 返回值
}
```

**`Append` 没有 error 返回值**，这意味着：

1. SQLite store 的 `json.Marshal` 失败时（`sqlite_store.go:291`），`payloadJSON` 为 nil，INSERT 写入空数据，**entry 静默丢失**
2. SQLite INSERT 本身失败时（磁盘满、锁超时、数据库损坏），**entry 静默丢失**
3. agent 循环中约 10+ 处调用 `Append`（`loop.go:76, 118, 119, 234, 235` 等），全部无法感知写入失败
4. **磁盘满的场景**：agent 继续执行工具（可能产生破坏性操作），但 tape 已停止记录。审计日志出现断裂，工具执行不可追溯

这不是"缺少错误处理"的问题，而是**接口设计层面的缺陷** — 所有实现都无法向上传递错误。

## TapeAgents 如何避免这个问题

TapeAgents 的 tape 是内存对象，`append()` 返回新 Tape 实例，不涉及 I/O，所以不会失败：

```python
def append(self, step: StepType) -> Self:
    return self.model_copy(update=dict(steps=self.steps + [step], metadata=TapeMetadata()))
```

持久化是异步观察者模式（`observe.py`）— 即使 SQLite 写入失败，tape 对象本身是完整的。失败只影响持久化，不影响 agent 运行时状态。

`execute_agent()` 的顶层 try/except 保证无论发生什么错误，都返回一个带有 `metadata.error` 标注的 tape。

## 修复方案

### 阶段 1：接口变更（必须做）

```go
type TapeStore interface {
    ListTapes() []string
    Reset(tape string)
    FetchAll(tape string, opts *FetchOpts) ([]TapeEntry, error)
    Append(tape string, entry TapeEntry) error  // ← 增加 error 返回
}
```

所有实现（InMemoryTapeStore、SQLiteTapeStore、FileTapeStore、PgStore）对应修改。

### 阶段 2：Agent 循环处理 Append 错误

```go
// pkg/agent/loop.go
if err := a.tape.Store.Append(tapeName, entry); err != nil {
    // 关键工具结果丢失 → 停止 agent，防止在无审计状态下继续执行
    log.Printf("[ERROR] tape append failed: %v", err)
    return &Result{Error: fmt.Errorf("tape write failure: %w", err)}
}
```

**策略选择**：
- 工具结果写入失败 → **停止 agent**（不能在无审计的情况下继续执行危险操作）
- 辅助性写入失败（event、usage 统计）→ **记录警告，继续运行**

### 阶段 3：写入缓冲（可选，参考 TapeAgents 的 SQLiteWriterThread）

```go
type BufferedTapeStore struct {
    inner   TapeStore
    buffer  []pendingEntry
    mu      sync.Mutex
    errCh   chan error
}

func (b *BufferedTapeStore) Append(tape string, entry TapeEntry) error {
    // 先写入内存 buffer（不会失败）
    b.mu.Lock()
    b.buffer = append(b.buffer, pendingEntry{tape, entry})
    b.mu.Unlock()
    
    // 异步刷到后端
    go b.flush()
    return nil
}
```

这样即使后端暂时不可用，agent 仍有最近的操作记录在内存中。

## 影响范围

- `pkg/tape/store.go` — 接口定义
- `pkg/tape/sqlite_store.go` — SQLite 实现
- `pkg/tape/inmemory_store.go` — 内存实现
- `pkg/tape/file_store.go` — 文件实现
- `pkg/tape/pg_store.go` — PostgreSQL 实现
- `pkg/tape/manager.go` — TapeManager
- `pkg/agent/loop.go` — Agent 循环（约 10 处调用）
- `plugins/cron/cron.go` — Cron 插件
- `plugins/tape/tape.go` — Tape 插件
- 所有测试文件

这是一个**破坏性变更**，需要一次性完成。

## 参考

- TapeAgents observe 模式：`tapeagents/observe.py`（持久化与运行时分离）
- TapeAgents execute_agent 错误保障：`tapeagents/orchestrator.py:308-329`
