# P0: 确定性重放测试（Deterministic Replay Testing）

> 灵感来源：TapeAgents `ReplayLLM` + `replay_tape()`

## 问题

**DMR 的 agent 循环（`pkg/agent/loop.go`）没有任何集成测试。**

当前 `pkg/agent/` 下 7 个测试文件全部测试工具函数：

| 测试文件 | 测试内容 | 是否调用 Agent.Run() |
|----------|---------|---------------------|
| `subagent_test.go` | 参数验证 | 否（nil ChatClient） |
| `usage_test.go` | token 提取 | 否 |
| `tool_result_trunc_test.go` | 截断逻辑 | 否 |
| `compact_optimize_test.go` | 消息优化 | 否 |
| `compact_prompt_test.go` | 标签提取 | 否 |
| `preemptive_compact_test.go` | 阈值判断 | 否 |
| `reactive_handoff_test.go` | 错误分类 | 否 |

`reactive_handoff_test.go` 中甚至有注释："Testing with actual errors would require creating mock errors"。

这意味着：
- 修改 compact 逻辑可能悄悄破坏工具调用流程，无法发现
- 修改 prompt 模板无法验证 agent 行为是否符合预期
- CI 只测试了边角逻辑，核心路径完全裸奔

## TapeAgents 的做法

### ReplayLLM（`tapeagents/llms/replay.py`）

1. 从 SQLite 加载历史 LLM 调用（prompt → output 映射）
2. 按 prompt JSON 精确匹配（`json.dumps(messages, sort_keys=True)`）
3. 命中则返回录制的 response，不调用真实 API
4. 未命中时用 Levenshtein 距离找最近的已知 prompt，输出逐消息 diff 诊断
5. 始终 fail（不做 fuzzy 返回），确保严格确定性

### replay_tape（`tapeagents/orchestrator.py`）

逐步对比新旧 tape 的每个 step，验证 agent 在相同输入下产生相同输出。

## DMR 的实现方案

### 1. RecordingProvider — 录制真实交互

```go
// pkg/core/recording.go

type RecordingProvider struct {
    inner    ChatProvider
    records  []RecordedCall
    mu       sync.Mutex
}

type RecordedCall struct {
    Request  ChatRequest  `json:"request"`
    Response ChatResponse `json:"response"`
}

func (r *RecordingProvider) CreateChatCompletion(ctx context.Context, req ChatRequest) (ChatResponse, error) {
    resp, err := r.inner.CreateChatCompletion(ctx, req)
    if err == nil {
        r.mu.Lock()
        r.records = append(r.records, RecordedCall{Request: req, Response: resp})
        r.mu.Unlock()
    }
    return resp, err
}

func (r *RecordingProvider) Save(path string) error {
    data, _ := json.MarshalIndent(r.records, "", "  ")
    return os.WriteFile(path, data, 0644)
}
```

### 2. ReplayProvider — 确定性重放

```go
// pkg/core/replay.go

type ReplayProvider struct {
    records  []RecordedCall
    index    map[string]ChatResponse  // key = canonical(request)
}

func (r *ReplayProvider) CreateChatCompletion(ctx context.Context, req ChatRequest) (ChatResponse, error) {
    key := canonicalKey(req)
    if resp, ok := r.index[key]; ok {
        return resp, nil
    }
    // 诊断：找最近的已知 request
    closest, distance := r.findClosest(key)
    return ChatResponse{}, fmt.Errorf(
        "replay miss: no matching request (closest distance=%d)\n--- expected ---\n%s\n--- got ---\n%s",
        distance, closest, key,
    )
}

func canonicalKey(req ChatRequest) string {
    // 移除不稳定字段（timestamp 等），JSON 序列化 messages + tools
    stable := StableRequest{Messages: req.Messages, Tools: req.Tools, Model: req.Model}
    data, _ := json.Marshal(stable)
    return string(data)
}
```

### 3. Agent 循环集成测试

```go
// pkg/agent/loop_test.go

func TestAgentLoop_SimpleToolCall(t *testing.T) {
    // 加载录制的交互
    replay := core.NewReplayProviderFromFile("testdata/simple_tool_call.json")
    
    // 构建 agent
    ag := buildTestAgent(replay)
    
    // 运行
    result, err := ag.Run(ctx, "default", "读取 /tmp/test.txt 的内容")
    require.NoError(t, err)
    
    // 断言 tape entries
    entries := ag.tape.Store.FetchAll("default", nil)
    assert.Equal(t, "message", entries[0].Kind)
    assert.Equal(t, "tool_call", entries[1].Kind)
    assert.Equal(t, "tool_result", entries[2].Kind)
    assert.Equal(t, "message", entries[3].Kind)
}

func TestAgentLoop_CompactTriggered(t *testing.T) {
    replay := core.NewReplayProviderFromFile("testdata/compact_trigger.json")
    ag := buildTestAgent(replay)
    ag.Config.MaxToken = 1000  // 低阈值触发 compact
    
    result, err := ag.Run(ctx, "default", "执行多步任务")
    require.NoError(t, err)
    
    // 验证 compact 被触发
    entries := ag.tape.Store.FetchAll("default", nil)
    hasAnchor := false
    for _, e := range entries {
        if e.Kind == "anchor" { hasAnchor = true }
    }
    assert.True(t, hasAnchor, "compact should have created an anchor")
}
```

### 4. 测试数据生成命令

```bash
# 录制一次真实交互（开发时运行一次）
DMR_RECORD_LLM=testdata/simple_tool_call.json dmr run "读取 /tmp/test.txt 的内容"

# 之后在 CI 中重放（无需 API key）
go test ./pkg/agent/ -run TestAgentLoop
```

### 实现步骤

1. 实现 `RecordingProvider` 和 `ReplayProvider`（约 200 行 Go）
2. 在 `core.LLMCore` 添加 `WrapWithRecording()` 方法
3. 手动录制 3-5 个典型场景的交互数据
4. 编写对应的集成测试
5. 加入 CI（`go test ./pkg/agent/`）

## 解决的问题

- **回归测试**：修改 compact/handoff/tool 逻辑后，CI 能发现破坏
- **Prompt 模板验证**：system prompt 变更后可验证 agent 行为不变
- **无 API Key CI**：重放模式不需要真实 LLM 调用
- **新模型兼容性**：切换模型后录制新 fixture，对比行为差异

## 代价与风险

- 录制数据需要维护（prompt 模板变更后需重新录制）
- canonicalKey 需要仔细设计，忽略不稳定字段（timestamp、request_id 等）
- 流式响应的录制/重放更复杂（可先只支持非流式）

## 参考

- TapeAgents `ReplayLLM`：`tapeagents/llms/replay.py`
- TapeAgents `CachedLLM`：`tapeagents/llms/cached.py`
- TapeAgents `replay_tape`：`tapeagents/orchestrator.py:415-527`
- DMR 现有 mock 基础：`pkg/client/chat_test.go` 的 `fakeClient`
