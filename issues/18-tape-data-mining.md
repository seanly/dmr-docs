# P2: Tape 数据挖掘与 Prompt 优化

> 灵感来源：TapeAgents `Agent.reuse()` + `make_training_data()` + Demo 优化

## TapeAgents 的做法

TapeAgents 的核心洞察：**每次成功执行的 tape 都是一份高质量的 prompt-response 训练样本**。

### 1. Agent.reuse() — 从历史 tape 重建 prompt-response 对

```python
# tapeagents/agent.py:783
def reuse(self, tape: TapeType) -> tuple[TapeType, list[LLMCall]]:
    """从一条已完成的 tape 中，重建每个 step 的 (prompt, output) LLMCall 对"""
    # 遍历 tape 的每一步：
    # 1. past_tape = tape[:i] — 截取到当前步之前的 tape
    # 2. prompt = current_agent.make_prompt(past_tape) — 重建 prompt
    # 3. output = current_agent.make_llm_output(tape, i) — 合成 LLM output
    # 4. 验证：用重建的 output 重新跑 run_iteration，确认生成的 steps 一致
    #    如果不一致 → 抛出 TapeReuseFailure
    return new_tape, llm_calls
```

关键特性：不仅重建 prompt-output 对，还**验证**重建的结果与原始执行一致（通过 `_is_step_data_equal()` 逐步比对）。这意味着任何成功执行的 tape 都可以**可靠地回溯出 LLM 在每一步看到了什么 prompt、产生了什么 output**。

### 2. make_training_data() — 自动生成微调数据

```python
def make_training_data(self, tape: TapeType) -> list[TrainingText]:
    texts = []
    for prompt, steps in self.reuse(tape):
        # 把 prompt 的 messages 变成 text
        # 标记 n_predicted（哪些 token 是 LLM 生成的，需要被训练）
        texts.append(TrainingText(text=..., n_predicted=...))
    return texts
```

### 3. Demo 优化 — 从成功 tape 中提取 few-shot 示例

```python
# 成功的 tape → 提取关键步骤 → 作为 few-shot demo 注入后续 prompt
agent.optimize_demos(successful_tapes)
```

## DMR 可以学什么

DMR 的 tape 存储了完整的 agent 执行轨迹（user message → tool_call → tool_result → assistant → ...）。这些数据目前**只用于上下文窗口管理和审计**，完全没有被二次利用。

### 方案：`dmr tape analyze` — Tape 数据挖掘 CLI

#### 1. 成功模式提取

```go
// pkg/tape/analyzer.go

type ExecutionPattern struct {
    ToolSequence []string          // 工具调用序列: ["search", "read_file", "edit"]
    AvgSteps     int               // 平均步骤数
    SuccessRate  float64           // 成功率
    ExampleTapes []string          // 代表性 tape 名称
}

// 从历史 tape 中提取成功的工具调用模式
func AnalyzePatterns(store TapeStore, filter AnalysisFilter) []ExecutionPattern {
    tapes := store.ListTapes()
    patterns := map[string]*ExecutionPattern{}
    
    for _, tapeName := range tapes {
        entries, _ := store.FetchAll(tapeName, nil)
        if !isSuccessfulExecution(entries) {
            continue
        }
        
        seq := extractToolSequence(entries)
        key := strings.Join(seq, "→")
        if p, ok := patterns[key]; ok {
            p.AvgSteps = (p.AvgSteps + len(entries)) / 2
            p.ExampleTapes = append(p.ExampleTapes, tapeName)
        } else {
            patterns[key] = &ExecutionPattern{
                ToolSequence: seq,
                AvgSteps:     len(entries),
                SuccessRate:  1.0,
                ExampleTapes: []string{tapeName},
            }
        }
    }
    return sortByFrequency(patterns)
}
```

#### 2. Prompt 效果分析

```go
// 分析不同 system prompt 版本下的 agent 表现
type PromptEffectiveness struct {
    PromptHash     string
    TotalRuns      int
    SuccessRuns    int
    AvgSteps       int
    AvgTokenCost   int
    CommonFailures []string
}

func AnalyzePromptEffectiveness(store TapeStore) []PromptEffectiveness {
    // 按 system prompt 的 hash 分组
    // 对比不同版本的成功率、步骤数、token 消耗
}
```

#### 3. 失败模式检测

```go
// 从失败的 tape 中提取常见错误模式
type FailurePattern struct {
    ErrorKind    string   // "tool_timeout", "context_overflow", "parse_error"
    LastToolCall string   // 失败前最后一次工具调用
    Frequency    int
    ExampleTapes []string
}

func DetectFailurePatterns(store TapeStore) []FailurePattern {
    // 分析失败的 tape，找出重复出现的错误模式
    // 帮助用户改进 system prompt 或工具配置
}
```

### CLI 接口

```bash
# 查看工具调用模式统计
dmr tape analyze patterns --since 7d

# 对比两个 prompt 版本的效果
dmr tape analyze prompts --compare v1.2..v1.3

# 检测失败模式
dmr tape analyze failures --top 10

# 导出成功 tape 为训练数据格式（未来扩展）
dmr tape export --format jsonl --filter success --output training.jsonl
```

## 为什么值得做

| 维度 | 说明 |
|------|------|
| **解决什么问题** | 用户无法从历史执行中学习；prompt 优化全靠直觉 |
| **ROI** | 中等（需要新的 analyzer 包，但利用现有 tape 数据，无需额外存储） |
| **可扩展性** | 可以逐步扩展：模式分析 → prompt 对比 → 训练数据导出 → 自动 prompt 优化 |
| **与现有架构** | 只读操作，不影响现有 tape 写入路径 |
| **DMR 特有价值** | DMR 作为 DevOps agent，长期运行积累大量 tape；挖掘这些数据的价值远高于研究框架 |

## 参考

- TapeAgents `Agent.reuse()`：`tapeagents/agent.py`
- TapeAgents `make_training_data()`：`tapeagents/agent.py`
- TapeAgents Demo 优化：`tapeagents/nodes.py` (StandardNode)
- TapeAgents 教师-学生蒸馏：`examples/gsm8k_tuning/`
- DMR Tape 存储：`pkg/tape/store.go`, `pkg/tape/manager.go`
