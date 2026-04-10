# P1: LLM 调用观测与审计（LLM Call Observability）

> 灵感来源：TapeAgents `observe.py` listener 模式 + SQLite 持久化

## 问题

DMR 的 tape 记录了消息和工具调用，但**不记录原始的 LLM API 交互**。

当前 tape 中的 `message` entry 存的是处理后的助手文本，你无法回答：
- "这次调用实际发了多少 prompt tokens？"
- "哪次 LLM 调用最贵？"
- "agent 发给 LLM 的完整 prompt 长什么样？"
- "过去一周的 LLM 成本是多少？"

`StreamEvents` 中有 `StreamUsage` 事件，`usage_test.go` 能追踪 token 用量，但这些数据是**临时的**，不持久化，也不可查询。

## TapeAgents 的做法

### observe.py 的 listener 模式

```python
# 模块级 listener 列表
llm_call_listeners: list[Callable] = [sqlite_store_llm_call]
tape_listeners: list[Callable] = [sqlite_store_tape]

# 每次 LLM 调用后触发
def observe_llm_call(call: LLMCall):
    for listener in llm_call_listeners:
        listener(call)
```

### SQLite schema

```sql
CREATE TABLE LLMCalls (
    prompt_id TEXT PRIMARY KEY,
    timestamp TEXT,
    prompt TEXT,              -- 完整 prompt JSON
    output TEXT,              -- 完整 response JSON
    prompt_length_tokens INT,
    output_length_tokens INT,
    cached INTEGER,
    llm_info TEXT,            -- model_name, parameters, context_size
    cost REAL
);
```

### 后台写入

`SQLiteWriterThread` 用 `queue.Queue` 异步写入，不阻塞 agent 主循环。

## DMR 的实现方案

### 作为插件实现（`plugins/llmaudit/`）

不侵入核心代码，通过 hook 系统接入。

```go
// plugins/llmaudit/llmaudit.go

type Plugin struct {
    config Config
    db     *sql.DB
    ch     chan AuditEntry
}

type Config struct {
    Enabled  bool   `toml:"enabled"`
    Level    string `toml:"level"`    // "metadata" | "full" | "none"
    DBPath   string `toml:"db_path"`  // 默认 ~/.dmr/llm_audit.db
    Retain   string `toml:"retain"`   // 数据保留期 "30d"
}

type AuditEntry struct {
    Timestamp        time.Time
    TapeName         string
    RunID            string
    Model            string
    PromptTokens     int
    CompletionTokens int
    TotalTokens      int
    Cost             float64           // 估算成本
    DurationMs       int64
    Prompt           json.RawMessage   // level="full" 时填充
    Response         json.RawMessage   // level="full" 时填充
    Cached           bool
    Error            string
}
```

### Hook 接入点

```go
func (p *Plugin) RegisterHooks(reg *plugin.HookRegistry) {
    // 方案 A：使用 AfterAgentRun hook
    reg.Register("AfterAgentRun", plugin.CallAll, p.onAfterRun)
    
    // 方案 B：需要核心新增 hook（更细粒度）
    // reg.Register("AfterLLMCall", plugin.CallAll, p.onLLMCall)
}
```

当前 `AfterAgentRun` hook 能拿到的信息有限。更理想的方案是在 `core.LLMCore` 或 `client.ChatClient` 中新增一个 `AfterLLMCall` 回调点：

```go
// pkg/core/execution.go 新增

type LLMCallRecord struct {
    Model            string
    PromptTokens     int
    CompletionTokens int
    DurationMs       int64
    Cached           bool
    Error            error
    // level="full" 时填充
    Request          *ChatRequest
    Response         *ChatResponse
}

type LLMCallObserver func(record LLMCallRecord)

// 在 RunChat / RunChatStream 结束时调用
func (c *LLMCore) notifyObservers(record LLMCallRecord) {
    for _, obs := range c.observers {
        obs(record)
    }
}
```

### SQLite Schema

```sql
CREATE TABLE llm_calls (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp   TEXT NOT NULL,
    tape_name   TEXT NOT NULL,
    run_id      TEXT,
    model       TEXT NOT NULL,
    prompt_tokens     INTEGER,
    completion_tokens INTEGER,
    total_tokens      INTEGER,
    cost_usd    REAL,
    duration_ms INTEGER,
    cached      INTEGER DEFAULT 0,
    error       TEXT,
    prompt      TEXT,     -- NULL unless level="full"
    response    TEXT      -- NULL unless level="full"
);

CREATE INDEX idx_llm_calls_tape ON llm_calls(tape_name);
CREATE INDEX idx_llm_calls_model ON llm_calls(model);
CREATE INDEX idx_llm_calls_date ON llm_calls(timestamp);
```

### 查询命令

```bash
# 成本汇总
dmr audit summary                    # 最近 7 天总览
dmr audit summary --by model         # 按模型分组
dmr audit summary --by tape          # 按 tape 分组
dmr audit summary --since 2026-04-01 # 指定时间段

# 详细记录
dmr audit calls --tape "session-1" --last 20
dmr audit calls --model "gpt-4o" --costly  # 按成本排序

# 导出
dmr audit export --format csv --since 2026-04-01
```

### 成本估算

```go
// 内置主流模型定价表（可配置覆盖）
var defaultPricing = map[string]ModelPricing{
    "gpt-4o":           {PromptPer1K: 0.0025, CompletionPer1K: 0.01},
    "gpt-4o-mini":      {PromptPer1K: 0.00015, CompletionPer1K: 0.0006},
    "claude-sonnet-4-6": {PromptPer1K: 0.003, CompletionPer1K: 0.015},
    // ...
}
```

### 隐私考量

- **`level="metadata"`（默认）**：只记录 token 数、模型、耗时、成本，不记录内容
- **`level="full"`**：记录完整 prompt 和 response，仅建议开发环境使用
- **数据保留**：`retain="30d"` 自动清理旧记录
- **敏感数据**：prompt 中可能包含用户数据，`level="full"` 需明确启用

## 解决的问题

- **成本控制**：企业环境中追踪和预算 LLM 使用成本
- **调试**：理解"LLM 到底看到了什么 prompt"（`level="full"` 模式）
- **模型对比**：同一任务用不同模型的 token 消耗和效果对比
- **异常检测**：发现异常高 token 消耗（可能是 prompt 膨胀 bug）

## 实现步骤

1. 在 `core.LLMCore` 新增 `LLMCallObserver` 回调接口（约 30 行）
2. 实现 `plugins/llmaudit/` 插件（约 400 行）
3. 新增 `dmr audit` 子命令（约 200 行）
4. 默认配置 `enabled: false`，用户显式启用

## 参考

- TapeAgents `observe.py`：`tapeagents/observe.py`
- TapeAgents `LLMCall` 数据模型：`tapeagents/core.py:378`
- DMR 现有 token tracking：`pkg/agent/usage_test.go`
- DMR `AfterAgentRun` hook：`pkg/plugin/hooks.go`
