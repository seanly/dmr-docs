# P1: 会话管理增强 — Tape 列表、清理与恢复

## 问题

DMR 使用 tape 作为会话单位，但当前缺乏基本的会话管理能力：

1. **无法列出已有 tape**：`/tape <name>` 可以切换 tape，但没有 `/tape list` 查看所有 tape。用户必须记住自己创建过哪些会话名，或者去 SQLite 数据库里手动查询
2. **无法删除/清理过期 tape**：长期使用后 tape 会不断累积，没有清理机制。SQLite 文件持续膨胀
3. **无法查看 tape 状态**：当前 tape 有多少条 entry、最后一次使用时间、消耗了多少 token——全部不可见
4. **启动时不恢复上次会话**：`dmr chat` 每次启动默认进入新 tape，之前中断的对话需要手动 `/tape <name>` 恢复，且用户往往忘了 tape 名称
5. **Brain 模式下的 tape 更难管理**：多 channel 并发创建 tape，没有全局视图

### 现状

```go
// plugins/cli/session.go:164
func (cs *CLIChatSession) handleCommand(line string) (handled, quit bool) {
    // 只支持 /tape <name> 切换，无 list/delete/info
    case strings.HasPrefix(line, "/tape "):
        parts := strings.Fields(line)
        if len(parts) > 1 {
            cs.tape = parts[1]
            cs.renderer.PrintInfo(fmt.Sprintf("Switched to tape: %s", cs.tape))
        }
}
```

## 设计方案

### 1. 新增 CLI 会话命令

```text
/tape              — 显示当前 tape 名称和基本信息
/tape list         — 列出所有 tape（名称、entry 数、最后活跃时间）
/tape <name>       — 切换到指定 tape（已有行为）
/tape info [name]  — 显示指定 tape 详细信息（entry 数、anchor 数、token 估算、创建时间）
/tape delete <name> — 标记删除 tape（软删除，保留 30 天后自动清理）
/tape gc           — 立即清理已标记删除的 tape
```

### 2. TapeStore 扩展

```go
// pkg/tape/store.go — 新增方法

type TapeSummary struct {
    Name        string    `json:"name"`
    EntryCount  int       `json:"entry_count"`
    AnchorCount int       `json:"anchor_count"`
    FirstEntry  time.Time `json:"first_entry"`
    LastEntry   time.Time `json:"last_entry"`
    SizeBytes   int64     `json:"size_bytes"`   // 仅 SQLite/File store
}

// ListTapes 返回所有 tape 的摘要信息
func (s *SQLiteStore) ListTapes(ctx context.Context) ([]TapeSummary, error)

// DeleteTape 软删除（标记 deleted_at）
func (s *SQLiteStore) DeleteTape(ctx context.Context, name string) error

// GarbageCollect 清理超过 retention 天数的已删除 tape
func (s *SQLiteStore) GarbageCollect(ctx context.Context, retentionDays int) (int, error)
```

### 3. 启动时恢复提示

```go
// plugins/cli/session.go — runREPL 开始时

func (cs *CLIChatSession) suggestResumeSession() {
    tapes, _ := cs.tapeStore.ListTapes(ctx)
    // 找最近活跃的非默认 tape
    var recent *TapeSummary
    for _, t := range tapes {
        if t.Name != "cli:main" && t.LastEntry.After(time.Now().Add(-24*time.Hour)) {
            if recent == nil || t.LastEntry.After(recent.LastEntry) {
                recent = &t
            }
        }
    }
    if recent != nil {
        cs.renderer.PrintInfo(fmt.Sprintf(
            "Last session: %s (%d entries, %s ago). Use /tape %s to resume.",
            recent.Name, recent.EntryCount, 
            time.Since(recent.LastEntry).Round(time.Minute),
            recent.Name,
        ))
    }
}
```

### 4. 自动清理策略

在 `dmr brain` 启动时或作为 cron 任务：

```toml
# ~/.dmr/config.toml
[tape]
auto_gc = true
retention_days = 90    # 已删除 tape 保留天数
max_entries = 100000   # 单 tape 最大 entry 数（超出时建议 compact）
```

## 实现步骤

- [ ] 在 `pkg/tape/` 的各 Store 实现中添加 `ListTapes()`、`DeleteTape()`、`GarbageCollect()` 方法
- [ ] 扩展 CLI 的 `/tape` 命令系列（list/info/delete/gc）
- [ ] 添加启动时恢复建议逻辑
- [ ] 在 `tapeInfo` 工具中复用 `ListTapes()` 能力
- [ ] 添加 `[tape] auto_gc` 配置项
- [ ] Brain 模式下的定期 GC 调度

## 代价与风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| 误删活跃 tape | 低 | 高 | 软删除 + 30 天保留期，不立即物理删除 |
| ListTapes 在大量 tape 时性能 | 中 | 低 | SQL 分页查询，CLI 默认显示最近 20 条 |
| PostgreSQL store 的 GC 实现差异 | 中 | 低 | 各 store 独立实现，接口统一 |
| 恢复提示在自动化场景中干扰 | 低 | 低 | 非 TTY 时跳过提示 |

## 参考

- DMR `pkg/tape/sqlite_store.go` — 当前 SQLite 存储实现
- DMR `plugins/cli/session.go` — 当前 `/tape` 命令处理
- DMR `plugins/tape/tape.go` — tapeInfo/tapeSearch 工具
