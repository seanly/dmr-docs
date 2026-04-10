# P3: Tape 浏览器与人工审查标注

> 灵感来源：TapeAgents Studio 的 tape 浏览/parent-child 导航 + Renderer 层级 + observe.py 的 listener 标注模式

## TapeAgents 的做法

### 1. Tape 浏览器 — 导航与检索

TapeAgents 的 SQLite 存储（`observe.py`）支持按 metadata 查询 tape：

```python
# tapeagents/observe.py

class SQLiteObserver:
    """将 tape 持久化到 SQLite，支持查询和 listener 回调"""
    
    def get_tapes(self, filter=None):
        """按 agent_name, status, parent_id 等条件查询"""
        pass
    
    def get_tape_by_id(self, tape_id):
        """获取单条 tape 及其所有 step"""
        pass
```

Studio UI 提供了 tape 列表 → 点击查看 → parent/child 导航的完整浏览体验。

### 2. Parent-Child 导航

通过 `TapeMetadata.parent_id` 字段，可以在 tape 之间跳转：

```python
class TapeMetadata(BaseModel):
    id: str
    parent_id: str | None  # 父 tape（调用者）
    author: str            # 哪个 agent 执行的
    
# 浏览时可以：
# tape_A (parent) → tape_B (subagent) → tape_C (sub-subagent)
#                 ← 返回 parent
```

### 3. Renderer 层级 — 按场景选择展示方式

```python
# tapeagents/renderers/

class BasicRenderer:
    """纯文本：适合 CLI/日志"""

class PrettyRenderer(BasicRenderer):
    """结构化 HTML：折叠/展开、语法高亮、step 类型图标"""

class CameraReadyRenderer(PrettyRenderer):
    """发布级：嵌入图片/视频、CSS 美化、可导出 PDF"""
```

### 4. Listener 模式 — 标注与回调

```python
# 当新 tape 被写入时，触发回调
observer.add_listener(lambda tape: annotate(tape))
```

## DMR 可以学什么

DMR 的 tape 目前**只有写入和窗口化读取**（`TapeManager.ReadMessages`），没有：
- 浏览所有 tape 的方式（除了直接查 SQLite）
- Parent-child 导航（Issue #02 已提出 lineage，但没有浏览器）
- 人工标注能力
- 结构化渲染

### 方案：`dmr tape` CLI 增强 + 标注系统

#### 1. Tape 列表与搜索

```go
// cmd/tape.go

// dmr tape list — 列出所有 tape
// dmr tape list --since 24h --agent devops-agent --status completed
func listTapes(store TapeStore, filter TapeFilter) {
    tapes := store.ListTapes()
    for _, name := range tapes {
        entries, _ := store.FetchAll(name, &FetchOpts{Limit: 1})
        meta := extractMeta(entries)
        
        fmt.Printf("%-30s  %s  %d steps  %s\n",
            name,
            meta.CreatedAt.Format("2006-01-02 15:04"),
            meta.TotalSteps,
            statusBadge(meta.Status),
        )
    }
}

// dmr tape show <name> — 查看单条 tape
// dmr tape show <name> --step 5 — 查看第 5 步
// dmr tape show <name> --kind tool_call — 只看工具调用
func showTape(store TapeStore, name string, opts ShowOpts) {
    entries, _ := store.FetchAll(name, nil)
    renderer := selectRenderer(opts.Format)
    
    for i, entry := range entries {
        if opts.Kind != "" && entry.Kind != opts.Kind {
            continue
        }
        fmt.Println(renderer.Render(i, entry))
    }
}

// dmr tape tree <name> — 展示 tape 的 parent-child 树
func showTapeTree(store TapeStore, name string) {
    // 依赖 Issue #02 的 lineage 机制
    lineage := extractLineage(store, name)
    printTree(lineage, 0)
    // 输出：
    // main-task-20260410
    //   ├── subagent-search-1
    //   ├── subagent-deploy-2
    //   │   └── subagent-verify-3
    //   └── subagent-notify-4
}
```

#### 2. 人工标注系统

```go
// pkg/tape/annotation.go

type Annotation struct {
    ID        string    `json:"id"`
    TapeName  string    `json:"tape_name"`
    EntryID   int       `json:"entry_id"`     // 标注在哪个 entry 上
    Author    string    `json:"author"`       // 标注者
    Label     string    `json:"label"`        // 标签: "correct", "wrong", "needs_review"
    Comment   string    `json:"comment"`      // 详细评论
    CreatedAt time.Time `json:"created_at"`
}

type AnnotationStore interface {
    Add(ann Annotation) error
    ListByTape(tapeName string) ([]Annotation, error)
    ListByLabel(label string) ([]Annotation, error)
}
```

存储在 tape 同一个 SQLite 数据库中（新增 `annotations` 表）：

```sql
CREATE TABLE annotations (
    id TEXT PRIMARY KEY,
    tape_name TEXT NOT NULL,
    entry_id INTEGER,            -- NULL = 标注整条 tape
    author TEXT NOT NULL,
    label TEXT NOT NULL,
    comment TEXT,
    created_at TEXT NOT NULL,
    FOREIGN KEY (tape_name) REFERENCES tapes(name)
);

CREATE INDEX idx_ann_tape ON annotations(tape_name);
CREATE INDEX idx_ann_label ON annotations(label);
```

#### 3. CLI 标注命令

```bash
# 给整条 tape 打标签
dmr tape annotate <tape-name> --label correct --comment "Successfully resolved the incident"

# 给特定步骤打标签
dmr tape annotate <tape-name> --step 5 --label wrong --comment "Should not have deleted the config"

# 查看标注
dmr tape annotations <tape-name>

# 按标签过滤 tape
dmr tape list --label needs_review
dmr tape list --label wrong --since 7d
```

#### 4. 审查工作流

```go
// pkg/tape/review.go

type ReviewWorkflow struct {
    store      TapeStore
    annStore   AnnotationStore
}

// 标记需要审查的 tape
func (w *ReviewWorkflow) FlagForReview(tapeName string, reason string) error {
    return w.annStore.Add(Annotation{
        TapeName: tapeName,
        Label:    "needs_review",
        Comment:  reason,
        Author:   "system",
    })
}

// 自动标记规则（Hook 插件）
func (w *ReviewWorkflow) AutoFlag(entry TapeEntry) {
    // 规则 1：执行了高危命令 → 自动标记审查
    if isHighRiskTool(entry) {
        w.FlagForReview(entry.TapeName, "high-risk tool used: "+entry.Payload["name"].(string))
    }
    
    // 规则 2：agent 循环了 → 标记审查
    // 规则 3：token 消耗异常高 → 标记审查
}

// 审查队列
func (w *ReviewWorkflow) GetReviewQueue() []ReviewItem {
    anns, _ := w.annStore.ListByLabel("needs_review")
    // 按优先级排序、去重
    return buildReviewQueue(anns)
}
```

#### 5. 与数据挖掘（Issue #18）的协同

标注数据可以反哺数据挖掘：

```go
// 只分析被标注为 "correct" 的 tape → 更高质量的模式提取
func AnalyzeVerifiedPatterns(store TapeStore, annStore AnnotationStore) []ExecutionPattern {
    correctTapes := annStore.ListByLabel("correct")
    return AnalyzePatterns(store, AnalysisFilter{
        TapeNames: correctTapes,
    })
}
```

## 为什么值得做

| 维度 | 说明 |
|------|------|
| **解决什么问题** | 团队无法系统性地审查 agent 行为；无法积累"什么是好的执行"的知识 |
| **DMR 独特性** | DMR 面向企业 DevOps 团队，人工审查是安全合规的硬性要求 |
| **tape list/show 成本** | 极低（利用现有 TapeStore API，新增 CLI 命令） |
| **标注系统成本** | 低-中（SQLite 新表 + 简单 CRUD） |
| **审查工作流成本** | 中（自动标记规则 + Hook 集成） |
| **可扩展性** | CLI 标注 → Web UI 标注 → 团队协作审查 → 标注数据反哺优化 |

## 依赖关系

- Issue #02（Tape Lineage）→ parent-child 导航的基础
- Issue #18（Tape Data Mining）→ 标注数据反哺分析
- Issue #20（Explainable Action）→ 审查时看到 reasoning

## 参考

- TapeAgents Studio 浏览器：`tapeagents/studio.py`
- TapeAgents SQLite Observer：`tapeagents/observe.py`
- TapeAgents Renderer 层级：`tapeagents/renderers/`
- TapeAgents TapeMetadata：`tapeagents/core.py`
- DMR TapeStore：`pkg/tape/store.go`
- DMR SQLite Store：`pkg/tape/sqlite_store.go`
