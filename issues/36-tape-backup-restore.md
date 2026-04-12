# P2: Tape 备份与恢复 — 基于 Cloudflare R2 的归档方案

## 问题

DMR 的 tape 是核心审计数据，但当前存储层缺乏系统性的备份与恢复能力：

1. **本地单点故障**：SQLite（默认）和 file store 都存储在本地磁盘，磁盘损坏即数据全丢
2. **无增量备份**：唯一的"备份"行为是 `file_store.go:Reset()` 中的 `.bak` 重命名，仅防误删
3. **无远程归档**：随着 tape 持续增长，旧 entries 占用本地存储但很少被查询
4. **无跨环境迁移**：从 SQLite 迁移到 PostgreSQL（或反向）没有标准化的导入导出通道
5. **训练数据导出缺基础设施**：issue #18/#25/#27 均提出了 `tape export` 需求，但没有底层的归档存储来承接

## 为什么选择 Cloudflare R2

| 维度 | R2 | S3 | 本地备份 |
|------|-----|-----|----------|
| 出站流量费 | **$0** | $0.09/GB | $0 |
| 存储成本 | $0.015/GB-month | $0.023/GB-month | 磁盘成本 |
| 免费额度 | 10 GB + 1M 写 + 10M 读 | 无 | N/A |
| API 兼容性 | S3 兼容 | 原生 | N/A |
| Go SDK | aws-sdk-go-v2 直接可用 | 同左 | os 标准库 |

R2 最大优势：**零出站费**。恢复操作（从 R2 下载）不产生额外费用，这对灾难恢复场景至关重要。

## 设计方案

### 架构概览

```
                        ┌─────────────────────────────┐
                        │   Cloudflare R2 Bucket      │
                        │   dmr-backup/               │
                        │   ├── {workspace}/          │
                        │   │   ├── {tape}/           │
                        │   │   │   ├── full/         │
                        │   │   │   │   └── 20260412.jsonl.gz
                        │   │   │   ├── incr/         │
                        │   │   │   │   ├── 20260412-001500.jsonl.gz
                        │   │   │   │   └── 20260412-023000.jsonl.gz
                        │   │   │   └── manifest.json │
                        │   │   └── ...               │
                        │   └── ...                   │
                        └──────────────┬──────────────┘
                                       │ S3 API (PutObject / GetObject)
                                       │
┌──────────────┐    backup/restore    ┌┴──────────────┐
│  TapeStore   │◄────────────────────►│  R2Archiver   │
│  (SQLite/PG) │    FetchAll/Append   │               │
└──────────────┘                      └───────────────┘
```

### 核心类型

```go
// pkg/tape/archiver.go

// ArchiverConfig holds R2/S3-compatible storage configuration.
type ArchiverConfig struct {
    Endpoint        string // R2: https://<account_id>.r2.cloudflarestorage.com
    Bucket          string // e.g. "dmr-backup"
    AccessKeyID     string
    SecretAccessKey  string
    Workspace       string // 命名空间隔离
    Compress        bool   // gzip 压缩（默认 true）
}

// R2Archiver handles tape backup and restore via S3-compatible storage.
// 它不实现 TapeStore 接口 — 它是 TapeStore 的辅助工具，不是替代品。
type R2Archiver struct {
    client    *s3.Client
    bucket    string
    workspace string
    compress  bool
}
```

### 备份流程

```go
// BackupOpts controls what to back up.
type BackupOpts struct {
    Tape        string // 指定 tape，空则备份所有
    AfterID     int    // 增量备份：只备份 ID > AfterID 的 entries
    Full        bool   // 强制全量备份
}

// Backup exports tape entries to R2.
// 返回备份的 entry 数量和最大 ID（用于下次增量备份）。
func (a *R2Archiver) Backup(ctx context.Context, store TapeStore, opts BackupOpts) (*BackupResult, error) {
    tapes := []string{opts.Tape}
    if opts.Tape == "" {
        tapes = store.ListTapes()
    }

    for _, tape := range tapes {
        fetchOpts := &FetchOpts{}
        if opts.AfterID > 0 && !opts.Full {
            fetchOpts.AfterID = opts.AfterID  // 增量：只取新 entries
        }

        entries, err := store.FetchAll(tape, fetchOpts)
        if err != nil {
            return nil, fmt.Errorf("fetch tape %q: %w", tape, err)
        }
        if len(entries) == 0 {
            continue
        }

        // 序列化为 JSONL
        var buf bytes.Buffer
        writer := a.newWriter(&buf) // gzip or plain
        for _, e := range entries {
            data, _ := json.Marshal(e)
            writer.Write(data)
            writer.Write([]byte("\n"))
        }
        writer.Close()

        // 上传到 R2
        key := a.objectKey(tape, opts)  // e.g. "ws/tape-name/incr/20260412-143000.jsonl.gz"
        _, err = a.client.PutObject(ctx, &s3.PutObjectInput{
            Bucket: &a.bucket,
            Key:    &key,
            Body:   bytes.NewReader(buf.Bytes()),
        })
        if err != nil {
            return nil, fmt.Errorf("upload %q: %w", key, err)
        }

        // 更新 manifest
        a.updateManifest(ctx, tape, entries, key)
    }
    return result, nil
}
```

### 恢复流程

```go
// RestoreOpts controls how to restore.
type RestoreOpts struct {
    Tape       string    // 指定 tape
    TargetTape string    // 恢复到不同名称（可选，默认同名）
    Before     time.Time // 恢复到指定时间点（基于备份时间戳）
    DryRun     bool      // 只输出统计，不实际写入
}

// Restore imports tape entries from R2 into a TapeStore.
func (a *R2Archiver) Restore(ctx context.Context, store TapeStore, opts RestoreOpts) (*RestoreResult, error) {
    // 1. 读取 manifest，确定需要下载的 objects
    manifest := a.loadManifest(ctx, opts.Tape)
    objects := manifest.ObjectsBefore(opts.Before) // 全量 + 增量序列

    // 2. 按顺序下载并解析
    var allEntries []TapeEntry
    for _, obj := range objects {
        resp, err := a.client.GetObject(ctx, &s3.GetObjectInput{
            Bucket: &a.bucket,
            Key:    &obj.Key,
        })
        if err != nil {
            return nil, fmt.Errorf("download %q: %w", obj.Key, err)
        }

        reader := a.newReader(resp.Body) // gzip or plain
        scanner := bufio.NewScanner(reader)
        for scanner.Scan() {
            var e TapeEntry
            json.Unmarshal(scanner.Bytes(), &e)
            allEntries = append(allEntries, e)
        }
    }

    // 3. 去重（增量备份可能有重叠）
    allEntries = deduplicateByOriginalID(allEntries)

    if opts.DryRun {
        return &RestoreResult{EntryCount: len(allEntries), DryRun: true}, nil
    }

    // 4. 写入目标 store
    targetTape := opts.TargetTape
    if targetTape == "" {
        targetTape = opts.Tape
    }
    for _, e := range allEntries {
        if err := store.Append(targetTape, e); err != nil {
            return nil, fmt.Errorf("append entry %d: %w", e.ID, err)
        }
    }

    return &RestoreResult{EntryCount: len(allEntries)}, nil
}
```

### Manifest 结构

每个 tape 在 R2 上维护一个 `manifest.json`，记录备份历史：

```json
{
  "tape": "session-abc123",
  "workspace": "my-project",
  "backups": [
    {
      "key": "my-project/session-abc123/full/20260410.jsonl.gz",
      "type": "full",
      "time": "2026-04-10T02:00:00Z",
      "entry_count": 5200,
      "min_id": 1,
      "max_id": 5200,
      "size_bytes": 1048576
    },
    {
      "key": "my-project/session-abc123/incr/20260411-140000.jsonl.gz",
      "type": "incremental",
      "time": "2026-04-11T14:00:00Z",
      "entry_count": 300,
      "min_id": 5201,
      "max_id": 5500,
      "size_bytes": 65536
    }
  ],
  "last_full_backup": "2026-04-10T02:00:00Z",
  "last_entry_id": 5500
}
```

### CLI 接口

```bash
# 全量备份所有 tapes
dmr tape backup --target r2

# 增量备份指定 tape
dmr tape backup --tape session-abc123 --incremental

# 查看备份列表
dmr tape backup list
# Output:
#   session-abc123  5,500 entries  2 backups  last: 2026-04-11T14:00Z
#   session-def456  1,200 entries  1 backup   last: 2026-04-10T02:00Z

# 恢复到当前 store
dmr tape restore --tape session-abc123

# 恢复到不同名称（不覆盖现有 tape）
dmr tape restore --tape session-abc123 --as session-abc123-recovered

# 恢复到指定时间点
dmr tape restore --tape session-abc123 --before "2026-04-11T00:00:00Z"

# Dry run：查看恢复统计
dmr tape restore --tape session-abc123 --dry-run
# Output:
#   Would restore 5,200 entries from 1 full + 0 incremental backups
```

### 配置

```toml
# ~/.dmr/config.toml

[tape.backup]
driver = "r2"                  # 或 "s3"（同一实现，不同 endpoint）
endpoint = "https://<account_id>.r2.cloudflarestorage.com"
bucket = "dmr-backup"
access_key_id = "..."          # 或引用 credential store
secret_access_key = "..."
compress = true                # gzip 压缩（默认）

# 自动备份（通过 cron 插件）
[tape.backup.schedule]
full = "0 2 * * 0"            # 每周日凌晨 2 点全量
incremental = "0 */6 * * *"   # 每 6 小时增量
retention_days = 90            # 90 天后自动清理旧备份
```

## 关键设计决策

### 1. 为什么不实现 TapeStore 接口

R2 是对象存储，不适合作为 append-heavy 的主存储：
- 每次 Append 需要 read-modify-write 整个 object
- 没有原生的查询/过滤能力
- 无文件锁，并发 Append 不安全

因此 `R2Archiver` 是 TapeStore 的**辅助工具**，不是第五种 store 实现。

### 2. 增量备份依赖 AfterID

利用 `FetchOpts.AfterID`（已有字段，[store.go:28](../../dmr/pkg/tape/store.go#L28)），只取上次备份后的新 entries。这比基于时间的增量更精确，因为 entry ID 是单调递增的。

### 3. 恢复时 ID 重新生成

通过 `Append` 写入目标 store 时，ID 由目标库分配。原始 ID 保留在 JSONL 中（`TapeEntry.ID` 字段），用于去重和审计，但不用于目标库的主键。

anchor 引用基于 name（不是 ID），因此 anchor 语义在恢复后完全保留。

### 4. Gzip 压缩

tape 数据是高度可压缩的 JSON 文本。实测 JSONL 的 gzip 压缩比通常在 5:1 ~ 10:1，显著降低存储成本和传输时间。

## 成本估算

### 场景 A：个人开发者（10 tapes，每 tape ~10K entries，~2KB/条）

| 项目 | 月用量 | R2 费用 |
|------|--------|---------|
| 原始数据量 | ~200 MB | - |
| 压缩后存储 | ~30 MB | **$0**（免费 10 GB 内） |
| 全量备份 PutObject | 4 次/月 | **$0**（免费 1M 次内） |
| 增量备份 PutObject | ~120 次/月 | **$0** |
| 恢复 GetObject | 偶尔 | **$0**（免费 10M 次内） |
| **月总计** | | **$0** |

### 场景 B：团队 / 多 Agent（100 tapes，每 tape ~100K entries）

| 项目 | 月用量 | R2 费用 |
|------|--------|---------|
| 原始数据量 | ~20 GB | - |
| 压缩后存储 | ~3 GB | **$0**（免费 10 GB 内） |
| 全量备份 PutObject | 400 次/月 | **$0** |
| 增量备份 PutObject | ~12,000 次/月 | **$0** |
| **月总计** | | **$0** |

### 场景 C：大规模生产（1000 tapes，每 tape ~1M entries，保留 1 年）

| 项目 | 月用量 | R2 费用 |
|------|--------|---------|
| 压缩后存储 | ~300 GB | ~$4.50/月 |
| 操作 | ~500K/月 | **$0** |
| 恢复出站 | - | **$0** |
| **月总计** | | **~$4.50** |

## 与现有 Issue 的关系

| Issue | 关系 |
|-------|------|
| #18 Tape 数据挖掘 | `dmr tape export` 可以直接从 R2 归档读取，避免查询主库 |
| #25 自演进架构 | 训练数据导出（tape → JSONL）可复用备份的 JSONL 格式 |
| #27 Tape 增强提案 | `export-training` 命令可以基于 R2 归档实现 |
| #14 SQLite Store 问题 | 冷数据归档到 R2 后，SQLite 主库体积减小，FTS 迁移压力降低 |
| #01 LLM 观测审计 | 审计数据的长期保留通过 R2 归档实现，不受本地存储限制 |

## 实现步骤

### Phase 1：核心备份恢复（~2 天）

- [ ] 新建 `pkg/tape/archiver.go`，实现 `R2Archiver`
- [ ] 实现 `Backup()`：全量 + 增量，gzip 压缩，manifest 管理
- [ ] 实现 `Restore()`：下载、解析、去重、写入
- [ ] 单元测试（使用 MinIO 或 mock S3）

### Phase 2：CLI 集成（~1 天）

- [ ] `dmr tape backup` 命令
- [ ] `dmr tape backup list` 命令
- [ ] `dmr tape restore` 命令（含 `--dry-run`、`--as`、`--before`）
- [ ] 配置文件解析（`[tape.backup]` section）

### Phase 3：自动化（~0.5 天）

- [ ] 与 cron 插件集成，支持 `[tape.backup.schedule]` 定时备份
- [ ] 过期清理：按 `retention_days` 自动删除旧备份
- [ ] 备份失败告警（日志 + 可选 webhook）

### Phase 4：冷归档与清理（~1 天）

- [ ] `dmr tape archive --older-than 30d`：将旧 entries 备份到 R2 后从主库删除
- [ ] 透明查询：archive 后的数据在 `dmr tape query` 时自动从 R2 拉取（lazy load）
- [ ] R2 Infrequent Access 存储类别支持（$0.01/GB-month，适合 >90 天数据）

## 参考

- DMR TapeStore 接口：`pkg/tape/store.go`
- DMR file store JSONL 格式：`pkg/tape/file_store.go`
- DMR FetchOpts.AfterID：`pkg/tape/store.go:28`
- Cloudflare R2 定价：$0.015/GB-month，零出站费，10 GB 免费
- Cloudflare R2 S3 API 兼容：支持 aws-sdk-go-v2
