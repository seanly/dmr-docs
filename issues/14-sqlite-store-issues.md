# P1: SQLite Store 读写锁误用 + FTS 迁移竞争

## 问题 1：读操作使用写锁

`pkg/tape/sqlite_store.go:261-263, 308-309`：

```go
func (s *SQLiteTapeStore) ListTapes() []string {
    s.mu.Lock()    // ← 应该是 RLock
    defer s.mu.Unlock()
}

func (s *SQLiteTapeStore) FetchAll(tape string, opts *FetchOpts) ([]TapeEntry, error) {
    s.mu.Lock()    // ← 应该是 RLock
    defer s.mu.Unlock()
}
```

读操作使用排他锁，序列化了所有读写。在 Brain 多通道模式下（多个 tape 并发读写），这是严重的性能瓶颈。所有通道的读操作互相阻塞。

**修复**：改为 `s.mu.RLock()` / `s.mu.RUnlock()`。

## 问题 2：FTS5 后台迁移竞争

`sqlite_store.go:88-101`：

```go
go func() {
    time.Sleep(3 * time.Second)
    if err := store.completeMigrationWithRetry(3); err != nil {
        ...
    } else {
        store.mu.Lock()
        store.migrationCompleted = true  // ← 写入标记
        store.mu.Unlock()
    }
}()
```

两个问题：
1. `migrationCompleted` 在其他代码路径中**无锁读取**（数据竞争）
2. `migrateWithTimeout` 超时后，后台 goroutine 仍在运行，同时**另一个**后台迁移被启动（`lines 83-101`），两个 goroutine 并发写 FTS 表

**修复**：
- `migrationCompleted` 改为 `atomic.Bool`
- `migrateWithTimeout` 使用 `context.WithCancel`，超时时取消 goroutine
- 或使用 `sync.Once` 确保只启动一次迁移

## 问题 3：FTS 迁移 NOT-IN 是 O(n^2)

`sqlite_store.go:184`：

```sql
SELECT id FROM entries WHERE id NOT IN (SELECT rowid FROM entries_fts) LIMIT ?
```

每批次都扫描整个 `entries_fts` 表。对于大数据库（百万级 entries），迁移时间会指数级增长。

**修复**：改为跟踪最后迁移的 ID：

```sql
SELECT id, ... FROM entries WHERE id > ? ORDER BY id LIMIT ?
```

## 参考

- TapeAgents 的 SQLite 写入：`tapeagents/observe.py`（后台线程 + 队列，避免锁竞争）
- DMR SQLite store：`pkg/tape/sqlite_store.go`
