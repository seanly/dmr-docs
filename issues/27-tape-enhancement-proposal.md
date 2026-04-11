# P1: Tape 增强提案 — 元数据、LLM 追踪、不可变操作与 ViewStack（Tape Enhancement Proposal）

> 灵感来源：TapeAgents `Tape` / `TapeMetadata` / `StepMetadata` / `TapeViewStack` / `LLMCall` / `Agent.reuse()`
>
> [TapeAgents](https://github.com/ServiceNow/TapeAgents) 是 Python 的 LLM Agent 框架，核心创新是 **Tape-First 架构**。与 DMR 类似使用 Tape 作为会话核心抽象，但在元数据、不可变性、LLM 追踪和多 Agent 隔离方面有更精致的设计。本提案分析其设计优点，提出 DMR 五项改进方案。

## 问题

DMR 的 Tape 系统存在四个结构性不足：

### 1. Step 元数据简单

```go
// DMR 当前
type TapeEntry struct {
    ID      int
    Kind    string          // "message", "tool_call", "event"...
    Payload map[string]any  // 实际内容
    Meta    map[string]any  // 扩展字段（无结构）
    Date    string
}
```

- 无法追踪 Entry 由哪个 Prompt/Agent/Node 产生
- 无法进行性能分析（哪个环节慢）
- 无法精确统计 Token 使用

### 2. 命令式 API

```go
// 直接修改存储
tm.AppendEntry(tapeName, entry)
```

- 无法在内存中操作 Tape
- 难以实现分支/合并
- 需要依赖存储层实现重放

### 3. LLM 调用记录不完整

当前只记录 Usage：

```go
eventData["usage"] = opts.Usage  // 只有 prompt_tokens, completion_tokens
```

- 无法生成训练数据
- 无法审计 Prompt 内容
- 无法进行成本归因

### 4. 多 Agent 上下文管理简单

使用 Anchor 做上下文窗口：

```go
ctx := tape.NewLastAnchorContext()  // 从最后一个 anchor 开始
messages, _ := tm.ReadMessages(tapeName, ctx)
```

- 不支持复杂的子 Agent 调用链
- Agent 间通信需要手动管理

## TapeAgents 的做法

### 核心数据模型

```python
class Tape(BaseModel, Generic[ContextType, StepType]):
    metadata: TapeMetadata    # 包含 id, parent_id, author, n_added_steps
    context: ContextType      # 泛型上下文（对话上下文、工具定义等）
    steps: List[StepType]     # Step 列表

class TapeMetadata(BaseModel):
    id: str                   # UUID（每次修改后重新生成）
    parent_id: str | None     # 上游 tape ID
    author: str | None        # 产生这次修改的 agent
    n_added_steps: int        # 上次运行新增的 steps 数
    result: Any               # 执行结果/错误信息
```

### Step 类型体系

```python
Step (基类)
├── Observation (外部输入)
│   ├── UserStep
│   ├── ToolResult
│   └── SystemStep
├── Thought (中间思考)
│   └── AssistantThought
├── Action (可执行动作)
│   ├── AssistantStep
│   ├── ToolCalls
│   └── FinalStep
└── ControlFlow (控制流)
    ├── Call          # 调用子 Agent
    ├── Respond       # 从子 Agent 返回
    ├── SetNextNode   # 设置下一个 Node
    └── Broadcast     # 广播消息
```

### Step 元数据（关键设计）

每个 Step 都有丰富的元数据，提供完整的血缘追踪：

```python
class StepMetadata(BaseModel):
    id: str           # Step 唯一标识
    prompt_id: str    # 产生这个 step 的 prompt ID
    node: str         # 产生这个 step 的 Node 名称
    agent: str        # Agent 全称 (如 "root/child/grandchild")
    llm: str          # 使用的 LLM 名称
    other: dict       # 扩展字段（执行时间、token 使用等）
```

### 不可变数据模型

TapeAgents 的 Tape 是**函数式不可变**的：

```python
new_tape = tape.append(step)        # 新 Tape 有新 ID
new_tape = tape + other_tape        # 合并两个 Tape
sub_tape = tape[0:5]                # 切片，metadata 重置
```

优势：线程安全、可缓存、可重放、支持分支。

### TapeViewStack：多 Agent 上下文隔离

将扁平的 Steps 转换为层级视图，每个 Agent 只能看到自己的 Steps：

```
Tape Steps: [U1, A1, Call(sub), A2, Respond, A3, Call(sub2), ...]
              ↓
TapeViewStack.compute(tape) → [
    TapeView(agent="root",      steps=[U1, A1, Call, A3, ...]),
    TapeView(agent="root/sub",  steps=[Call, A2, Respond]),
    TapeView(agent="root/sub2", steps=[Call, ...])
]
```

### LLM 调用追踪

完整记录每次 LLM 调用，并支持生成训练数据：

```python
class LLMCall(BaseModel):
    timestamp: str
    prompt: Prompt           # 完整 prompt（messages + tools）
    output: LLMOutput        # LLM 原始输出
    prompt_length_tokens: int
    output_length_tokens: int
    logprobs: list[TokenLogprob]
    cached: bool

def make_training_data(self, tape: Tape) -> list[TrainingText]:
    """从 Tape 提取训练数据用于 SFT/RLHF"""
    _, llm_calls = self.reuse(tape)
    return [self.make_training_text(call) for call in llm_calls]
```

## DMR 的实现方案

### 方案 1：Step 元数据增强（高优先级）

**目标**：完整的执行血缘追踪

```go
// 扩展现有 EntryMeta（保持向后兼容）
type EntryMeta struct {
    // 现有字段
    Scope string `json:"scope,omitempty"`
    
    // 新增字段
    PromptID   string     `json:"prompt_id,omitempty"`
    Agent      string     `json:"agent,omitempty"`
    LLM        string     `json:"llm,omitempty"`
    Node       string     `json:"node,omitempty"`
    ToolCallID string     `json:"tool_call_id,omitempty"`
    DurationMs int        `json:"duration_ms,omitempty"`
    TokenUsage TokenUsage `json:"token_usage,omitempty"`
}

type TokenUsage struct {
    PromptTokens     int `json:"prompt_tokens"`
    CompletionTokens int `json:"completion_tokens"`
    TotalTokens      int `json:"total_tokens"`
}
```

实施要点：
1. 修改 `tape.Entry` 结构（保持 JSON 序列化兼容）
2. 在 `agent.Run` 中记录 PromptID、Agent、LLM
3. 在 `ToolExecutor` 中记录 ToolCallID
4. 在 LLM 调用后记录 TokenUsage

### 方案 2：LLM 调用完整追踪（高优先级）

**目标**：支持训练数据生成和完整审计

```go
// 新增 Entry Kind: "llm_call"
type LLMCallPayload struct {
    Prompt   LLMPrompt   `json:"prompt"`
    Output   LLMOutput   `json:"output"`
    Usage    TokenUsage  `json:"usage"`
    Cached   bool        `json:"cached"`
    Cost     float64     `json:"cost,omitempty"`
    Duration int         `json:"duration_ms,omitempty"`
}

type LLMPrompt struct {
    Messages []Message       `json:"messages"`
    Tools    []ToolSchema    `json:"tools,omitempty"`
}

type LLMOutput struct {
    Content   string     `json:"content"`
    Reasoning string     `json:"reasoning,omitempty"`
    ToolCalls []ToolCall `json:"tool_calls,omitempty"`
}
```

训练数据生成：

```go
// pkg/tape/training.go
type TrainingText struct {
    Text        string
    NPredicted  int
    PromptText  string
    OutputText  string
}

func (m *TapeManager) MakeTrainingData(tapeName string) ([]TrainingText, error) {
    entries, err := m.Store.FetchAll(tapeName, &FetchOpts{Kinds: []string{"llm_call"}})
    if err != nil {
        return nil, err
    }
    
    var texts []TrainingText
    for _, entry := range entries {
        payload, ok := entry.Payload["prompt"].(LLMPrompt)
        if !ok {
            continue
        }
        promptText := formatMessages(payload.Messages)
        outputText := entry.Payload["output"].(LLMOutput).Content
        
        texts = append(texts, TrainingText{
            Text:       fmt.Sprintf("<|user|>\n%s<|end|>\n<|assistant|>\n%s<|end|>", promptText, outputText),
            NPredicted: len(outputText),
            PromptText: promptText,
            OutputText: outputText,
        })
    }
    return texts, nil
}
```

### 方案 3：内存中 Tape 操作（中优先级）

**目标**：支持内存中的 Tape 转换和分支

```go
// pkg/tape/tape.go

type Tape struct {
    Metadata TapeMetadata
    Context  map[string]any
    Entries  []TapeEntry
}

type TapeMetadata struct {
    ID            string
    ParentID      string
    Author        string
    NAddedEntries int
    CreatedAt     time.Time
}

// 函数式操作（返回新 Tape）

func (t *Tape) Append(entry TapeEntry) *Tape {
    return &Tape{
        Metadata: TapeMetadata{
            ID:            generateID(),
            ParentID:      t.Metadata.ID,
            Author:        entry.Meta.Agent,
            NAddedEntries: 1,
            CreatedAt:     time.Now(),
        },
        Context: t.Context,
        Entries: append(t.Entries, entry),
    }
}

func (t *Tape) Slice(start, end int) *Tape {
    return &Tape{
        Metadata: TapeMetadata{
            ID:       generateID(),
            ParentID: t.Metadata.ID,
        },
        Context: t.Context,
        Entries: t.Entries[start:end],
    }
}

func (t *Tape) Concat(other *Tape) *Tape {
    return &Tape{
        Metadata: TapeMetadata{
            ID:            generateID(),
            ParentID:      t.Metadata.ID,
            NAddedEntries: len(other.Entries),
        },
        Context: t.Context,
        Entries: append(t.Entries, other.Entries...),
    }
}

func (t *Tape) Filter(predicate func(TapeEntry) bool) *Tape {
    var filtered []TapeEntry
    for _, e := range t.Entries {
        if predicate(e) {
            filtered = append(filtered, e)
        }
    }
    return &Tape{
        Metadata: TapeMetadata{ID: generateID(), ParentID: t.Metadata.ID},
        Context:  t.Context,
        Entries:  filtered,
    }
}
```

与存储层集成：

```go
func (m *TapeManager) LoadTape(tapeName string) (*Tape, error) {
    entries, err := m.Store.FetchAll(tapeName, nil)
    if err != nil {
        return nil, err
    }
    return &Tape{
        Metadata: TapeMetadata{ID: tapeName},
        Entries:  entries,
    }, nil
}

func (m *TapeManager) SaveTape(tapeName string, tape *Tape) error {
    startIdx := len(tape.Entries) - tape.Metadata.NAddedEntries
    if startIdx < 0 {
        startIdx = 0
    }
    for i := startIdx; i < len(tape.Entries); i++ {
        m.Store.Append(tapeName, tape.Entries[i])
    }
    return nil
}
```

### 方案 4：ViewStack 多 Agent 上下文（中优先级）

**目标**：支持复杂的子 Agent 调用链和上下文隔离

```go
// pkg/tape/viewstack.go

type ViewStack struct {
    Stack           []TapeView
    MessagesByAgent map[string][]ControlFlowEntry
}

type TapeView struct {
    AgentName         string
    AgentFullName     string
    Entries           []TapeEntry
    EntriesByKind     map[string][]TapeEntry
    OutputsBySubagent map[string]TapeEntry
    LastNode          string
    NextNode          string
}

type ControlFlowEntry struct {
    Kind      string // "call", "respond", "broadcast"
    FromAgent string
    ToAgent   string
    Content   string
    EntryID   int
}

func ComputeViewStack(tape *Tape, rootAgent string) *ViewStack {
    vs := &ViewStack{
        Stack: []TapeView{{
            AgentName:     rootAgent,
            AgentFullName: rootAgent,
        }},
        MessagesByAgent: make(map[string][]ControlFlowEntry),
    }
    for _, entry := range tape.Entries {
        vs.Update(entry)
    }
    return vs
}
```

Agent 层级命名：

```
root                    # 根 Agent
root/analyst           # root 调用的 analyst 子 Agent
root/analyst/coder     # analyst 调用的 coder 子 Agent
```

与现有 Subagent 集成：

```go
func (p *SubagentPlugin) CallTool(req *proto.CallToolRequest, resp *proto.CallToolResponse) error {
    subagentName := req.Args["agent_name"].(string)
    prompt := req.Args["prompt"].(string)
    currentAgent := req.Context["agent_path"].(string)
    fullPath := currentAgent + "/" + subagentName
    
    p.tape.AppendEntry(req.TapeName, tape.NewCallEntry(subagentName, prompt, currentAgent))
    
    viewStack := tape.ComputeViewStack(p.tape.LoadTape(req.TapeName), "root")
    subView := viewStack.GetView(fullPath)
    result, err := p.runSubagent(fullPath, prompt, subView.BuildMessages())
    
    p.tape.AppendEntry(req.TapeName, tape.NewRespondEntry(result, fullPath, currentAgent))
    resp.Result = result
    return nil
}
```

### 方案 5：Tape 重用和验证（中优先级）

**目标**：Agent 行为回归测试和迁移学习

```go
// pkg/tape/reuse.go

type TapeReuseError struct {
    StepIndex   int
    Expected    TapeEntry
    Got         TapeEntry
    PartialTape *Tape
}

type ReuseConfig struct {
    StrictMode    bool
    AllowExtraLLM bool
}

func (a *Agent) ReuseTape(tape *Tape, cfg ReuseConfig) (*Tape, []LLMCallPayload, error) {
    var reusedEntries []TapeEntry
    var llmCalls []LLMCallPayload
    
    for i, entry := range tape.Entries {
        if !isAgentEntry(entry) {
            reusedEntries = append(reusedEntries, entry)
            continue
        }
        pastTape := &Tape{Entries: tape.Entries[:i]}
        prompt := a.MakePrompt(pastTape)
        originalCall := extractLLMCall(entry)
        newEntries := a.GenerateSteps(pastTape, originalCall.Output)
        
        if !entriesMatch(newEntries, entry) {
            if cfg.StrictMode {
                return nil, nil, &TapeReuseError{
                    StepIndex: i, Expected: entry, Got: newEntries[0],
                    PartialTape: &Tape{Entries: reusedEntries},
                }
            }
            reusedEntries = append(reusedEntries, entry)
        } else {
            reusedEntries = append(reusedEntries, newEntries...)
        }
        llmCalls = append(llmCalls, originalCall)
    }
    
    return &Tape{
        Metadata: TapeMetadata{ID: generateID(), ParentID: tape.Metadata.ID, Author: a.Name},
        Entries:  reusedEntries,
    }, llmCalls, nil
}
```

CLI 命令：

```bash
dmr tape verify --tape <name> --strict
dmr tape reuse --from <old-tape> --to <new-tape>
dmr tape export-training --tape <name> --output training.jsonl
```

## 实现步骤

### Phase 1：元数据增强

- [ ] 修改 `tape.EntryMeta` 结构，添加新字段
- [ ] 在 `agent.Run` 中记录 PromptID、Agent、LLM
- [ ] 在 `ToolExecutor` 中记录 ToolCallID
- [ ] 在 LLM 调用后记录 TokenUsage
- [ ] 更新 SQLite schema（如果 Meta 存储为 JSON，则无需修改）

### Phase 2：LLM 调用追踪

- [ ] 新增 `llm_call` Entry Kind
- [ ] 在 `core/execution.go` 中记录完整 LLM 调用
- [ ] 实现 `MakeTrainingData` 功能
- [ ] 添加 `dmr tape training-data` CLI 命令

### Phase 3：内存中 Tape 操作

- [ ] 创建 `pkg/tape/tape.go`，定义内存中 Tape 结构
- [ ] 实现不可变操作（Append, Slice, Concat, Filter）
- [ ] 添加 `TapeManager.LoadTape` 和 `SaveTape` 方法
- [ ] 重构现有代码使用新的 Tape 类型

### Phase 4：ViewStack

- [ ] 设计 Agent 层级命名规范
- [ ] 实现 `ComputeViewStack` 和相关类型
- [ ] 修改 `subagent` 插件使用 ViewStack
- [ ] 支持 Broadcast Entry Kind

### Phase 5：Tape 重用

- [ ] 实现 `ReuseTape` 函数
- [ ] 添加 `dmr tape verify` 和 `reuse` 命令
- [ ] 编写测试用例

## 解决的问题

- **可观测性**：完整的执行血缘追踪，精确的性能和成本分析
- **可训练性**：从生产 Tape 直接生成 SFT/RLHF 训练数据
- **可测试性**：Tape 重用机制支持 Agent 行为回归测试和迁移学习
- **可扩展性**：ViewStack 支持任意深度的子 Agent 调用链和上下文隔离

## 代价与风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| 向后兼容性问题 | 中 | 高 | EntryMeta 使用 JSON 序列化，新增字段不影响旧代码 |
| 性能下降（元数据记录） | 低 | 中 | 元数据记录是 O(1) 操作，影响可忽略 |
| 存储空间增加 | 中 | 低 | LLM 调用记录会增加存储，可配置是否启用 |
| 代码复杂度增加 | 中 | 中 | 分阶段实施，每个阶段独立可测试 |

## 参考

- TapeAgents `Tape` 核心定义：`tapeagents/core.py:285-335`
- TapeAgents `ViewStack` 实现：`tapeagents/view.py`
- TapeAgents `Agent.reuse()`：`tapeagents/agent.py:783-837`
- TapeAgents 训练数据生成：`tapeagents/agent.py:880-895`
- DMR `pkg/tape/entry.go` — Entry 定义
- DMR `pkg/tape/manager.go` — TapeManager
- DMR `pkg/agent/loop.go` — Agent Loop
- DMR `pkg/core/execution.go` — LLM 执行
