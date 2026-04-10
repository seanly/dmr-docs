# P2: Plan-Survey-Act 结构化推理模式

> 灵感来源：TapeAgents GAIA Agent 的 Plan→FactsSurvey→Act 自循环

## TapeAgents 的做法

GAIA Agent 是 TapeAgents 中最复杂的示例，实现了一个**结构化推理循环**：

### 1. Plan 阶段 — 制定行动计划

```python
# examples/gaia_agent/steps.py:45-51
class Plan(Thought):
    """Agent 输出一个分步计划（作为 Thought step 记录在 tape 中）"""
    plan: list[str]  # ["1. 搜索 X", "2. 读取 Y", "3. 计算 Z"]
```

Agent 首先输出一个计划，然后**按计划执行**。计划被记录在 tape 中，可以被后续步骤引用。

### 2. FactsSurvey — 自我评估已知/未知

```python
# examples/gaia_agent/steps.py:54-75
class FactsSurvey(Thought):
    """Agent 自检：将任务所需的事实分为四个类别"""
    kind: Literal["facts_survey_thought"] = "facts_survey_thought"
    given_facts: list[str] = Field(
        description="List of facts already provided in the question"
    )
    facts_to_lookup: list[str] = Field(
        description="List of facts that need to be found on the web or in documents"
    )
    facts_to_derive: list[str] = Field(
        description="List of facts that need to be derived from given facts using reasoning or code execution"
    )
    facts_to_guess: list[str] = Field(
        description="List of facts that need to be guessed from given facts, documents, and reasoning"
    )
```

在执行计划的过程中，Agent 定期做 FactsSurvey（作为 `Thought` step 记录在 tape 中）：
- 已知了什么？→ given_facts
- 还需要查什么？→ facts_to_lookup
- 需要推导什么？→ facts_to_derive
- 需要猜测什么？→ facts_to_guess

这让 agent 能**自我纠偏**，而不是盲目执行原始计划。

### 3. Act 阶段 — 带有 reasoning 的工具调用

每次工具调用都引用计划中的某一步，形成**计划→执行→验证**的闭环。

### 完整循环

```
Plan → Act(step 1) → FactsSurvey → 
  if unresolved: Act(step 2) → FactsSurvey →
    if all resolved: FinalAnswer
    if plan needs revision: Re-Plan → ...
```

## DMR 为什么需要这个

DMR 当前的 agent 循环是**纯反应式**的：

```
User Message → LLM Think → Tool Call → Tool Result → LLM Think → ...
```

没有显式的计划阶段，没有自检机制。对于复杂的 DevOps 任务（例如"排查生产环境的内存泄漏"），agent 容易：
1. **缺乏全局视角**：只看到当前步骤，不知道整体方向
2. **重复无效动作**：没有 FactsSurvey，不知道已经尝试了什么
3. **过早收敛**：一旦找到一个可能的原因就停止，不验证其他可能性

## 修复方案

### 1. Workflow Skill 扩展：Plan-Act-Verify 模板

```yaml
# skills/plan_act_verify.yaml
name: plan_act_verify
description: Structured reasoning for complex diagnostic tasks

phases:
  - name: plan
    prompt: |
      Analyze the user's request and create a numbered action plan.
      Output format:
      <plan>
      1. [action description]
      2. [action description]
      ...
      </plan>
    max_steps: 1
    
  - name: execute
    prompt: |
      Execute your plan step by step.
      After each tool call, assess:
      - What did I learn?
      - Does the plan need adjustment?
      - Am I making progress?
    max_steps: 20
    
  - name: verify
    prompt: |
      Review all findings:
      - Confirmed facts: [list]
      - Unresolved questions: [list]
      If unresolved questions remain, go back to execute phase.
      If all questions resolved, provide final answer.
    can_loop_to: execute  # 可以回到执行阶段
```

### 2. System Prompt 内嵌（更轻量，无需新基础设施）

```go
// pkg/agent/prompts.go

const structuredReasoningPrompt = `
## Structured Reasoning Protocol

For complex tasks, follow this protocol:

### Phase 1: Plan
Before taking any action, output your plan:
<plan>
1. [First step and why]
2. [Second step and why]
...
</plan>

### Phase 2: Execute with Self-Check
After every 3 tool calls, perform a facts survey:
<facts_survey>
Confirmed: [what you know for sure]
Unresolved: [what you still need to find out]
Plan status: [which steps are done, which remain]
</facts_survey>

### Phase 3: Conclude
Only provide a final answer when all plan steps are complete
or you've determined the remaining steps are unnecessary.
`
```

### 3. Tape 层面的支持 — 结构化 entry 类型

```go
// 新增 entry kinds（不需要修改 TapeEntry 结构）
const (
    KindPlan        = "plan"         // agent 输出的计划
    KindFactsSurvey = "facts_survey" // 自检结果
    KindPlanUpdate  = "plan_update"  // 计划修正
)

// 解析 LLM 输出中的结构化标签
func extractStructuredContent(assistantMsg string) []TapeEntry {
    var entries []TapeEntry
    
    if plan := extractTag(assistantMsg, "plan"); plan != "" {
        entries = append(entries, TapeEntry{
            Kind: KindPlan,
            Payload: map[string]any{
                "steps": parsePlanSteps(plan),
            },
        })
    }
    
    if survey := extractTag(assistantMsg, "facts_survey"); survey != "" {
        entries = append(entries, TapeEntry{
            Kind: KindFactsSurvey,
            Payload: map[string]any{
                "confirmed":  extractSection(survey, "Confirmed"),
                "unresolved": extractSection(survey, "Unresolved"),
                "plan_status": extractSection(survey, "Plan status"),
            },
        })
    }
    
    return entries
}
```

### 4. 与现有 compact 机制的协调

Plan 和 FactsSurvey entries 在 compact 时应该被**优先保留**（它们是 agent 的"工作记忆"）：

```go
// pkg/tape/manager.go — compact 时的优先级
func compactPriority(entry TapeEntry) int {
    switch entry.Kind {
    case "plan", "facts_survey":
        return 10  // 高优先级，最后被 compact
    case "tool_result":
        return 1   // 低优先级，优先被裁剪
    default:
        return 5
    }
}
```

## 为什么值得做

| 维度 | 说明 |
|------|------|
| **解决什么问题** | 复杂任务时 agent 缺乏全局规划，容易迷失方向或重复无效操作 |
| **最小实现** | 只改 system prompt（Phase 2 方案），零代码修改 |
| **完整实现** | Workflow Skill 模板 + 结构化 entry 类型 + compact 优先级 |
| **DMR 特有价值** | DevOps 故障排查是 Plan-Act-Verify 的完美场景（排查步骤有序、需要验证假设） |
| **可扩展性** | 结构化的 plan/facts_survey entries 可以用于：进度报告、审计、数据挖掘 |

## 参考

- TapeAgents GAIA Agent：`examples/gaia_agent/`
- TapeAgents PlanAction/FactsSurvey：`examples/gaia_agent/steps.py`
- DMR Agent Loop：`pkg/agent/loop.go`
- DMR Workflow Skill：（如已存在）
- DMR Compact 机制：`pkg/tape/manager.go`
