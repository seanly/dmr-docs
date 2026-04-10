# P1: 可解释动作 — 工具调用必须附带理由

> 灵感来源：TapeAgents `ExplainableAction` + GAIA Agent 的 reasoning 要求

## TapeAgents 的做法

### ExplainableAction — 强制工具调用带理由

```python
# tapeagents/steps.py:16-20
class ExplainableAction(Action):
    """Action that requires the agent to explain why it chose this tool"""
    reason_to_use: str = Field(
        description="A summary of the reason you want to use this tool written in the form of 'I want to'",
        default=""
    )
```

Agent 在调用工具时，不仅要指定工具名和参数，还需要填写 `reason_to_use` 字段解释**为什么选择这个工具**。通过 Pydantic `Field(description=...)` 将指令嵌入 JSON schema，LLM 在 function-calling 时会看到并遵循。虽然 `default=""` 不强制，但 schema 描述作为软约束引导 LLM 填写。这个字段被记录在 tape 中，用于：
1. **事后审计**：审查员可以看到 agent 的决策理由
2. **调试**：当 agent 选错工具时，可以看到它的"思考过程"
3. **微调数据**：reasoning 字段提供了高质量的 chain-of-thought 训练数据

### GAIA Agent 的实践

```python
# examples/gaia_agent/ — 每个 action 都有 reasoning
class GaiaAction(ExplainableAction):
    tool: str
    args: dict
    reasoning: str  # "我需要搜索这个信息因为..."
```

## DMR 为什么特别需要这个

DMR 定位为**审计驱动**的生产运行时。但当前的 tool_call 记录只有：

```json
{
  "kind": "tool_call",
  "payload": {
    "name": "exec_command",
    "arguments": {"command": "rm -rf /tmp/old-data"}
  }
}
```

审计员看到这条记录时，只知道 agent 执行了 `rm -rf`，但**不知道为什么**。在 DevOps 场景中：
- 安全审计需要知道**为什么** agent 选择删除这些文件
- 合规报告需要追溯决策链
- 事故回顾需要了解 agent 在"想什么"

## 修复方案

### 1. tool_call 条目增加 reasoning 字段

```go
// 不需要修改 TapeEntry 结构（payload 已经是 map[string]any）
// 只需要在 tool_call 的 payload 中增加 reasoning 字段

// pkg/agent/reactive_handoff.go 或工具调用组装处
func buildToolCallEntry(call ToolCall) TapeEntry {
    return TapeEntry{
        Kind: "tool_call",
        Payload: map[string]any{
            "name":      call.Name,
            "arguments": call.Arguments,
            "reasoning": call.Reasoning, // ← 新增
        },
    }
}
```

### 2. System Prompt 引导 LLM 输出 reasoning

```go
// 在 system prompt 中加入指令
const reasoningInstruction = `
When calling a tool, you MUST include a "reasoning" field explaining:
1. Why this tool is the right choice for the current step
2. What you expect the result to be
3. What you'll do next based on the result

Example:
{
  "name": "exec_command",
  "arguments": {"command": "df -h"},
  "reasoning": "Checking disk usage because the user reported slow writes. If usage > 90%, I'll suggest cleanup."
}
`
```

### 3. OPA 策略：高危工具必须有 reasoning

```rego
# policies/explainable_action.rego

package dmr.policy

# 高危工具列表
high_risk_tools := {"exec_command", "file_write", "deploy", "kubectl"}

deny[msg] {
    input.tool_call.name == high_risk_tools[_]
    not input.tool_call.reasoning
    msg := sprintf("High-risk tool '%s' requires reasoning field", [input.tool_call.name])
}

deny[msg] {
    input.tool_call.name == high_risk_tools[_]
    count(input.tool_call.reasoning) < 20  # reasoning 太短
    msg := sprintf("Reasoning for '%s' is too brief (min 20 chars)", [input.tool_call.name])
}
```

### 4. 审计报告中展示 reasoning

```go
// pkg/tape/report.go

type AuditEntry struct {
    Timestamp  string
    ToolName   string
    Arguments  map[string]any
    Reasoning  string   // 展示在审计报告中
    Result     string
    PolicyPass bool
}

func GenerateAuditReport(store TapeStore, tapeName string) []AuditEntry {
    entries, _ := store.FetchAll(tapeName, nil)
    var report []AuditEntry
    
    for _, e := range entries {
        if e.Kind == "tool_call" {
            report = append(report, AuditEntry{
                Timestamp: e.Date,
                ToolName:  e.Payload["name"].(string),
                Arguments: e.Payload["arguments"].(map[string]any),
                Reasoning: getStringOrDefault(e.Payload, "reasoning", "<no reasoning>"),
            })
        }
    }
    return report
}
```

## 实现分级

| 阶段 | 内容 | 成本 |
|------|------|------|
| **Phase 1** | System prompt 添加 reasoning 指令 + 记录到 tape | 极低（只改 prompt 和序列化） |
| **Phase 2** | OPA 策略：高危工具强制 reasoning | 低（新增 rego 规则） |
| **Phase 3** | 审计报告模板展示 reasoning | 中（新增报告生成器） |
| **Phase 4** | Reasoning 质量评分（LLM-as-judge） | 高（可选，远期） |

## 为什么值得做

| 维度 | 说明 |
|------|------|
| **解决什么问题** | 审计日志有"做了什么"但无"为什么"，不满足严格合规要求 |
| **DMR 独特性** | DMR 的 OPA 策略引擎 + 审计 tape 让 reasoning 的强制和记录天然可行 |
| **成本** | Phase 1 几乎零成本（只改 prompt），就能获得显著的审计价值 |
| **可扩展性** | reasoning 数据未来可用于 agent 行为分析、异常检测、合规报告自动化 |

## 参考

- TapeAgents `ExplainableAction`：`tapeagents/steps.py`
- TapeAgents GAIA Agent：`examples/gaia_agent/`
- DMR OPA 策略：`pkg/policy/`
- DMR Tape 审计：`pkg/tape/`
