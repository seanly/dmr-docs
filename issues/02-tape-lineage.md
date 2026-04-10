# P1: Tape 谱系追踪（Tape Lineage Tracking）

> 灵感来源：TapeAgents `TapeMetadata.parent_id` 谱系机制

## 问题

DMR 的 tape 之间没有结构化的因果关系。

当前 subagent 的子 tape 命名约定是 `parentTape:subagent:shortID`，cron 的是 `cron:jobID`。但这只是**字符串约定**，存在以下问题：

1. 无法查询"这条子 tape 是从哪条 tape 的哪个 entry 触发的"
2. 多层 subagent 嵌套时无法回溯完整调用链
3. cron 触发的 subagent 的链路无法追踪
4. Brain 多通道模式下，外部插件触发的 agent run 的源头不可追溯
5. 命名约定不强制执行 — 插件可以用任意命名，打破约定

## TapeAgents 的做法

```python
class TapeMetadata(BaseModel):
    id: str = Field(default_factory=lambda: str(uuid4()))
    parent_id: str | None = None  # ← 关键字段
    author: str | None = None
    n_added_steps: int = 0
    error: Any | None = None
    result: Any = {}
```

每次 `tape.append(step)` 返回新 tape 时自动生成新 `id`，可以构建完整的 tape 演化树。

TapeAgents 的方案是面向不可变 tape 对象的。DMR 的 tape 是可变的持久化日志，需要不同的实现。

## DMR 的实现方案

### 不改 tape 对象模型，利用已有的 Meta 字段

在 anchor entry 的 `Meta` 中记录因果关系：

```go
// 当 subagent 创建子 tape 时
func (a *Agent) RunSubagent(parentTape string, parentEntryID int, prompt string) {
    childTape := fmt.Sprintf("%s:subagent:%s", parentTape, shortID)
    
    anchor := tape.NewAnchorEntry("subagent/job/"+jobID, nil)
    // 在 Meta 中记录谱系信息
    anchor.Meta = map[string]any{
        "lineage": map[string]any{
            "parent_tape":     parentTape,
            "parent_entry_id": parentEntryID,
            "trigger":         "subagent",
        },
    }
    a.tape.Store.Append(childTape, anchor)
    // ...
}
```

```go
// 当 cron 触发时
func (p *Plugin) runJob(job Job) {
    anchor := tape.NewAnchorEntry("cron/job/"+job.ID, nil)
    anchor.Meta = map[string]any{
        "lineage": map[string]any{
            "parent_tape":     job.TapeName,
            "trigger":         "cron",
            "cron_job_id":     job.ID,
            "cron_schedule":   job.Schedule,
        },
    }
    // ...
}
```

```go
// 当 Brain 外部插件触发时
func (a *Agent) handleExternalTrigger(channel, tapeName, prompt string) {
    anchor := tape.NewAnchorEntry("external/"+channel, nil)
    anchor.Meta = map[string]any{
        "lineage": map[string]any{
            "trigger": "external",
            "channel": channel,
        },
    }
    // ...
}
```

### 查询接口

```go
// pkg/tape/lineage.go

type LineageInfo struct {
    TapeName     string
    AnchorName   string
    ParentTape   string
    ParentEntry  int
    Trigger      string  // "subagent" | "cron" | "external" | "user"
    Metadata     map[string]any
}

// 获取一条 tape 的直接父级
func (m *TapeManager) GetParentLineage(tape string) (*LineageInfo, error) {
    entries, _ := m.Store.FetchAll(tape, &FetchOpts{LastAnchor: true})
    for _, e := range entries {
        if e.Kind == "anchor" && e.Meta != nil {
            if lineage, ok := e.Meta["lineage"].(map[string]any); ok {
                return parseLineageInfo(tape, e, lineage), nil
            }
        }
    }
    return nil, nil // 根 tape，无父级
}

// 递归获取完整调用链
func (m *TapeManager) GetFullLineage(tape string) ([]LineageInfo, error) {
    var chain []LineageInfo
    current := tape
    for {
        info, err := m.GetParentLineage(current)
        if err != nil || info == nil {
            break
        }
        chain = append(chain, *info)
        current = info.ParentTape
    }
    return chain, nil
}
```

### CLI 命令

```bash
# 查看一条 tape 的谱系
dmr tape lineage "session-1:subagent:abc123"
# 输出:
# session-1:subagent:abc123
#   ← triggered by: subagent (entry #42 on session-1)
#     ← session-1
#       ← root (user initiated)

# 查看某条 tape 的所有子 tape
dmr tape children "session-1"
# 输出:
# session-1:subagent:abc123 (subagent, 2026-04-10 14:30)
# session-1:subagent:def456 (subagent, 2026-04-10 14:35)

# Brain 模式下查看所有活跃通道的调用树
dmr brain tree
```

## 解决的问题

- **调试多层 subagent**：当 subagent 的 subagent 失败时，快速定位调用链
- **Cron 执行追溯**：了解定时任务触发了哪些子任务
- **Brain 通道监控**：理解各通道之间的交互关系
- **审计合规**：完整的操作溯源链

## 代价与风险

- 只修改 anchor 创建点（subagent.go、cron.go、brain_service.go），约 3 处
- 不改 TapeEntry 结构，不改 TapeStore 接口
- 查询是读取 + 解析 Meta 字段，性能开销可忽略
- 向后兼容：旧 tape 的 anchor 没有 lineage Meta，`GetParentLineage` 返回 nil

## 可扩展性

高。Meta 字段是 `map[string]any`，未来可以添加更多谱系信息（执行时长、token 消耗摘要等）而不需要改 schema。

## 参考

- TapeAgents `TapeMetadata`：`tapeagents/core.py:258-278`
- DMR subagent 子 tape 创建：`pkg/agent/subagent.go`
- DMR cron tape 创建：`plugins/cron/cron.go:runJob()`
- DMR anchor 结构：`pkg/tape/entry.go:103-109`
