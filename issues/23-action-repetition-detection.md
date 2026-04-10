# P1: 动作重复检测与集成投票

> 灵感来源：TapeAgents `ToolCollectionEnvironment.loop_detected()` + GAIA Agent 的 `action_repetitions()` + 多数投票集成

## TapeAgents 的做法

### 1. 环境级循环检测

```python
# tapeagents/environment.py:231-247

def loop_detected(self, tape: Tape) -> bool:
    actions = last_actions(tape)
    all_actions_counter = defaultdict(int)
    for step in tape.steps:
        if isinstance(step, Action):
            all_actions_counter[step.llm_view()] += 1
    for last_action in actions:
        n_args = len(last_action.llm_dict())
        if n_args < 2:
            continue  # 跳过无参数的 action（只有 kind 字段）
        cnt = all_actions_counter[last_action.llm_view()]
        if cnt >= self.loop_warning_after_n_steps:
            logger.warning(f"Loop, action {last_action.kind} repeated {cnt} times")
            return True
    return False
```

检测到循环后，注入一条**用户级别的警告**到 tape 中：

```python
def react(self, tape):
    if self.loop_detection and self.loop_detected(tape):
        obs = UserStep(content=self.loop_warning)  # "You seem to be stuck..."
        return tape.append(obs)
```

**关键设计**：用 `llm_view()`（action 的完整序列化表示）作为比较 key，精确匹配相同工具 + 相同参数的重复调用。

### 2. Agent 级重复计数（GAIA Agent）

```python
# examples/gaia_agent/eval.py:150-156

def action_repetitions(tape: GaiaTape) -> int:
    unique_actions = {}
    for step in tape:
        if isinstance(step, (SearchAction, PythonCodeAction)):
            key = step.llm_view()
            unique_actions[key] = unique_actions.get(key, 0) + 1
    return max(unique_actions.values(), default=0)
```

当重复次数超过阈值（默认 3），强制终止 agent：

```python
# examples/gaia_agent/eval.py:133
if action_repetitions(tape) >= max_action_repetitions:
    break  # 停止执行
```

### 3. 多数投票集成 — 多次执行取最优

```python
# examples/gaia_agent/eval.py:159-196

def ensemble_results(all_tapes, oracle=False):
    """对同一任务的多次执行结果，用多数投票选出最佳"""
    for tapes in zip(*all_tapes):
        results = [tape.metadata.result for tape in tapes]
        most_common_idx = majority_vote(results)  # 多数投票
        best_tape = tapes[most_common_idx]
        # 统计：improved / degraded / added_result
```

**思路**：同一个任务执行 3-5 次，每次 LLM 可能给出不同路径，用多数投票选出最可能正确的结果。在 GAIA benchmark 上，这种集成显著提升了准确率。

## DMR 为什么需要这个

### 问题 1：Agent 陷入死循环

DMR 的 agent loop（`pkg/agent/loop.go`）没有任何动作重复检测。当 agent 陷入循环时：

```
Step 5: tool_call exec_command {"command": "kubectl get pods -n default"}
Step 6: tool_result: "No resources found"
Step 7: tool_call exec_command {"command": "kubectl get pods -n default"}  ← 完全相同
Step 8: tool_result: "No resources found"
Step 9: tool_call exec_command {"command": "kubectl get pods -n default"}  ← 还是一样
... 直到 maxSteps 耗尽
```

**后果**：浪费 LLM token、浪费时间、用户体验差。在 token 按量计费的生产环境中，这直接等于烧钱。

### 问题 2：无安全阀

当前 DMR 唯一的退出条件是 `maxSteps`。但 maxSteps 可能设得很高（例如 50 步），agent 在 3 步就开始循环时，仍要等到 50 步才停。

## 修复方案

### 1. 核心检测器

```go
// pkg/agent/loop_detection.go

type LoopDetector struct {
    actionCounts map[string]int   // action signature → 出现次数
    threshold    int              // 重复阈值（默认 3）
    warning      string           // 注入的警告信息
}

func NewLoopDetector(threshold int) *LoopDetector {
    return &LoopDetector{
        actionCounts: make(map[string]int),
        threshold:    threshold,
        warning:      "You are repeating the same action. Try a different approach.",
    }
}

// actionSignature 提取工具调用的唯一签名
func actionSignature(entry TapeEntry) string {
    if entry.Kind != "tool_call" {
        return ""
    }
    name, _ := entry.Payload["name"].(string)
    args, _ := json.Marshal(entry.Payload["arguments"])
    
    // hash 参数避免 key 过长
    h := sha256.Sum256(args)
    return fmt.Sprintf("%s:%x", name, h[:8])
}

// Check 检查最新的 action 是否构成循环
func (d *LoopDetector) Check(entry TapeEntry) LoopStatus {
    sig := actionSignature(entry)
    if sig == "" {
        return LoopStatus{Detected: false}
    }
    
    d.actionCounts[sig]++
    count := d.actionCounts[sig]
    
    if count >= d.threshold {
        return LoopStatus{
            Detected:    true,
            Action:      entry.Payload["name"].(string),
            Repetitions: count,
            Warning:     d.warning,
        }
    }
    return LoopStatus{Detected: false}
}

type LoopStatus struct {
    Detected    bool
    Action      string
    Repetitions int
    Warning     string
}
```

### 2. 集成到 Agent Loop

```go
// pkg/agent/loop.go

func (a *Agent) Run(ctx context.Context, tape string) error {
    detector := NewLoopDetector(3) // 3 次重复即告警
    
    for step := 0; step < a.maxSteps; step++ {
        // ... LLM 调用 ...
        
        for _, toolCall := range response.ToolCalls {
            entry := buildToolCallEntry(toolCall)
            
            // 循环检测
            status := detector.Check(entry)
            if status.Detected {
                if status.Repetitions >= a.maxRepetitions { // 例如 5 次
                    // 强制终止
                    log.Printf("[WARN] Agent terminated: %s repeated %d times",
                        status.Action, status.Repetitions)
                    return ErrLoopDetected
                }
                
                // 注入警告（让 LLM 自己纠偏）
                warningEntry := TapeEntry{
                    Kind: "system",
                    Payload: map[string]any{
                        "content": status.Warning,
                        "reason":  "loop_detected",
                        "action":  status.Action,
                        "count":   status.Repetitions,
                    },
                }
                store.Append(tape, warningEntry)
            }
            
            // 正常执行
            store.Append(tape, entry)
            result := executeTool(ctx, toolCall)
            store.Append(tape, result)
        }
    }
}
```

### 3. 分级响应策略

```go
// 不同重复次数的不同响应
func (d *LoopDetector) getResponse(count int) LoopResponse {
    switch {
    case count == d.threshold:
        return LoopResponse{
            Action:  "warn",
            Message: "You seem to be repeating the same action. Consider a different approach.",
        }
    case count == d.threshold + 1:
        return LoopResponse{
            Action:  "warn_strong",
            Message: "WARNING: You have repeated this action " + strconv.Itoa(count) + 
                " times. You MUST try a completely different strategy.",
        }
    case count >= d.threshold + 2:
        return LoopResponse{
            Action:  "terminate",
            Message: "Agent terminated due to action loop.",
        }
    default:
        return LoopResponse{Action: "none"}
    }
}
```

### 4. 配置化

```toml
# config.toml

[agent.loop_detection]
enabled = true
warn_after = 3        # 3 次重复后注入警告
terminate_after = 5   # 5 次重复后强制终止
ignore_tools = ["search"]  # 某些工具允许重复调用（如搜索）
warning_message = "You are repeating the same action. Try a different approach."
```

### 5. 集成投票（高级，可选）

```go
// pkg/agent/ensemble.go

type EnsembleRunner struct {
    agent    *Agent
    runs     int  // 执行次数（默认 3）
}

func (e *EnsembleRunner) RunWithVoting(ctx context.Context, task string) (string, error) {
    var results []EnsembleResult
    
    for i := 0; i < e.runs; i++ {
        tape := fmt.Sprintf("ensemble-%s-run-%d", task, i)
        err := e.agent.Run(ctx, tape)
        result := extractFinalAnswer(tape)
        results = append(results, EnsembleResult{
            Tape:   tape,
            Answer: result,
            Error:  err,
        })
    }
    
    // 多数投票
    best := majorityVote(results)
    return best.Answer, nil
}

func majorityVote(results []EnsembleResult) EnsembleResult {
    counts := map[string]int{}
    for _, r := range results {
        if r.Error == nil && r.Answer != "" {
            counts[r.Answer]++
        }
    }
    
    var bestAnswer string
    var bestCount int
    for answer, count := range counts {
        if count > bestCount {
            bestAnswer = answer
            bestCount = count
        }
    }
    
    for _, r := range results {
        if r.Answer == bestAnswer {
            return r
        }
    }
    return results[0]
}
```

## 为什么值得做

| 维度 | 说明 |
|------|------|
| **解决什么问题** | Agent 陷入死循环时无法自我恢复，浪费 token 和时间 |
| **紧迫性** | 高——循环是 LLM agent 最常见的失败模式之一 |
| **核心实现成本** | 低（循环检测器 ~100 行 Go，集成到 loop.go 约 20 行） |
| **集成投票成本** | 中（需要多次执行，token 成本 ×N） |
| **DMR 特有价值** | 生产环境中 agent 卡住 = 运维事件无人处理 = 真实损失 |
| **可扩展性** | 检测 → 警告 → 终止 → 集成投票，渐进式增强 |

## 参考

- TapeAgents 循环检测：`tapeagents/environment.py:231-247`
- TapeAgents 循环警告注入：`tapeagents/environment.py:221-225`
- GAIA Agent 重复计数：`examples/gaia_agent/eval.py:150-156`
- GAIA Agent 集成投票：`examples/gaia_agent/eval.py:159-196`
- DMR Agent Loop：`pkg/agent/loop.go`
