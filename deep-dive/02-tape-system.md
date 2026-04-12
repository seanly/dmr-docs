# Tape 系统深入分析

## 概述

Tape 是 DMR 的核心组件之一，提供追加式审计日志功能。它记录所有消息、工具调用、错误和使用情况作为结构化数据。

## 核心设计原则

1. **追加式 (Append-only)**: 一旦写入，记录不可修改
2. **结构化**: 所有记录都有明确的类型和格式
3. **可查询**: 支持全文搜索、时间范围过滤、类型过滤
4. **上下文窗口**: 通过 anchor 机制管理对话上下文

## 核心类型

### TapeEntry

```go
type TapeEntry struct {
    ID      int            `json:"id"`      // 条目 ID
    Kind    string         `json:"kind"`    // 类型: message, system, anchor, tool_call, tool_result, error, event, compact_summary
    Payload map[string]any `json:"payload"` // 载荷数据
    Meta    map[string]any `json:"meta,omitempty"` // 元数据
    Date    string         `json:"date"`    // ISO 8601 格式时间戳
}
```

### 条目类型 (Kinds)

| 类型 | 用途 | 示例 |
|------|------|------|
| `message` | 对话消息 | `{"role": "user", "content": "hello"}` |
| `system` | 系统提示 | `{"content": "You are a helpful assistant"}` |
| `anchor` | 上下文锚点 | `{"name": "session/start", "state": {...}}` |
| `tool_call` | 工具调用 | `{"calls": [{"id": "1", "function": {...}}]}` |
| `tool_result` | 工具结果 | `{"results": [...]}` |
| `error` | 错误记录 | `{"kind": "denied", "message": "..."}` |
| `event` | 事件 | `{"name": "handoff", "data": {...}}` |
| `compact_summary` | 压缩摘要 | `{"content": "summary text"}` |

## TapeStore 接口

```go
type TapeStore interface {
    ListTapes() []string
    Reset(tape string)
    FetchAll(tape string, opts *FetchOpts) ([]TapeEntry, error)
    Append(tape string, entry TapeEntry) error
}
```

### 实现类型

1. **InMemoryTapeStore**: 内存存储，用于测试
2. **FileStore**: 文件存储 (JSON Lines)
3. **SQLiteStore**: SQLite 存储，支持 FTS5 全文搜索
4. **PostgresStore**: PostgreSQL 存储，支持 tsvector

## 查询选项 (FetchOpts)

```go
type FetchOpts struct {
    AfterAnchor    string       // 在指定 anchor 之后
    LastAnchor     bool         // 在最后一个 anchor 之后
    BetweenAnchors [2]string    // 在两个 anchor 之间
    StartDate      string       // 开始日期
    EndDate        string       // 结束日期
    TextQuery      string       // 全文搜索
    Kinds          []string     // 类型过滤
    Limit          int          // 限制数量
    AfterID        int          // ID 游标分页
}
```

## TapeManager

提供高级操作：

```go
type TapeManager struct {
    Store TapeStore
}

// 读取消息 (从 anchor 上下文)
func (m *TapeManager) ReadMessages(tape string, ctx *TapeContext) ([]map[string]any, error)

// 追加条目
func (m *TapeManager) AppendEntry(tape string, entry TapeEntry) error

// 创建 handoff (anchor + event)
func (m *TapeManager) Handoff(tape, name string, state map[string]any) ([]TapeEntry, error)

// 记录完整对话
func (m *TapeManager) RecordChat(opts RecordChatOpts)

// 压缩 (创建摘要)
func (m *TapeManager) Compact(ctx context.Context, opts CompactOpts) ([]TapeEntry, error)
```

## 上下文窗口 (TapeContext)

```go
type TapeContext struct {
    AnchorMode AnchorSelector  // NoAnchor, LastAnchorS, NamedAnchor
    AnchorName string          // 当 AnchorMode == NamedAnchor
    Select     func([]TapeEntry, *TapeContext) []map[string]any
    State      map[string]any
}
```

上下文选择器：
- **NoAnchor**: 不过滤，读取所有
- **LastAnchorS**: 从最后一个 anchor 开始
- **NamedAnchor**: 从指定命名的 anchor 开始

## Handoff 机制

Handoff 是 Tape 的核心概念，用于标记对话阶段边界：

```go
// 创建 handoff
anchor := NewAnchorEntry(name, state)
event := NewEventEntry("handoff", map[string]any{"name": name, "state": state})

// 使用场景
- 开始新话题时标记边界
- 完成任务时总结
- 上下文管理：最新 anchor 后的条目成为"活动"上下文
```

## 压缩 (Compact)

当上下文过长时，DMR 会自动压缩：

1. 读取当前上下文 (从最后一个 anchor)
2. 调用 LLM 生成摘要
3. 创建 anchor (压缩标记)
4. 创建 compact_summary 条目
5. 创建 event 条目记录压缩事件

```go
func (m *TapeManager) Compact(ctx context.Context, opts CompactOpts) ([]TapeEntry, error) {
    // 1. 读取当前上下文
    entries, _ := m.Store.FetchAll(opts.Tape, &FetchOpts{LastAnchor: true})
    
    // 2. 转换为消息
    messages := tapeCtx.BuildMessages(entries)
    
    // 3. 调用 LLM 生成摘要
    summary, _ := opts.Summarizer(ctx, messages)
    
    // 4. 创建条目
    anchor := NewAnchorEntry(name, state)
    summaryEntry := NewCompactSummaryEntry(summary)
    event := NewEventEntry("compact", data)
    
    // 5. 追加到 tape
    return []TapeEntry{anchor, summaryEntry, event}, nil
}
```

## 存储后端对比

| 特性 | Memory | File | SQLite | PostgreSQL |
|------|--------|------|--------|------------|
| 持久化 | ❌ | ✅ | ✅ | ✅ |
| 全文搜索 | ❌ | ❌ | ✅ (FTS5) | ✅ (tsvector) |
| 并发 | ✅ | ⚠️ | ✅ | ✅ |
| 适用场景 | 测试 | 简单使用 | 生产 | 企业 |

## 时区支持

```go
// 设置时区
func SetTimezone(tzName string) error

// 获取当前时区
func GetTimezone() *time.Location
```

默认使用系统本地时区，可通过配置 `tape.timezone` 指定 IANA 时区名（如 "Asia/Shanghai"）。
