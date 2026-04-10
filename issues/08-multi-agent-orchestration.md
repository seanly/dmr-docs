# P2: 多 Agent 编排模式（Multi-Agent Orchestration Patterns）

> 灵感来源：TapeAgents `Chain`、`CallSubagent`、`TeamAgent`、`TapeViewStack`、控制流 Node 体系

## DMR 当前的多 Agent 能力

DMR 的多 agent 是**单层 subagent 工具调用**：

```
Parent Agent
  ├─ LLM 决定调用 subagent tool
  ├─ subagent(prompt="做X") → 子 tape 运行 → 返回文本
  ├─ subagent(prompt="做Y") → 子 tape 运行 → 返回文本  (可并行)
  └─ LLM 综合文本结果
```

**具体限制：**

1. **禁止嵌套**：`subagent.go:27-29` 硬编码 `strings.Contains(parentTape, ":subagent:")` 拒绝嵌套调用
2. **只返回文本**：`RunSubagent` 返回 `(res.Output, nil)` — opaque string
3. **无编排模式**：没有 Chain（顺序管道）、没有 Router（条件路由）、没有 Scatter-Gather（扇出汇聚）
4. **编排全靠 LLM**：是否调用 subagent、调用几次、如何组合结果 — 全由 LLM 自行决定
5. **无子 agent 间通信**：两个 subagent 之间不能直接传递数据

## TapeAgents 的多 Agent 体系

TapeAgents 提供了三层递进的编排能力：

### 层 1：Call/Respond 原语

```python
Call(agent_name="Coder", content="写一段代码")  # 调用子 agent
Respond(content="结果", copy_output=True)        # 子 agent 返回，输出拷贝到父
Broadcast(to=["A","B"], content="任务说明")       # 广播给多个子 agent
```

这些是 **tape 上的控制流步骤**，不是工具调用。框架根据这些步骤自动切换活跃 agent。

### 层 2：组合模式

**Chain（顺序管道）**：
```python
Chain.create(nodes=[
    CallSubagent(agent=FindNouns),       # 步骤1：找名词
    CallSubagent(agent=FindVerbs),       # 步骤2：找动词
    CallSubagent(agent=FilterIrregular,  # 步骤3：过滤，输入来自步骤2
                 inputs=(-1,)),
    CallSubagent(agent=PresentResults,   # 步骤4：汇总，输入来自步骤1和3
                 inputs=(-2, -1)),
])
```

**TeamAgent（LLM 驱动选择）**：
```python
TeamAgent.create_team_manager(
    subagents=[Coder, Executor, Reviewer],
    nodes=[
        BroadcastLastMessageNode(),  # 广播最新消息
        SelectAndCallNode(),         # LLM 选择下一个 agent
        RespondOrRepeatNode(),       # 终止或循环
    ]
)
```

### 层 3：控制流 Node

```python
ControlFlowNode     # 无 LLM 调用的路由节点
  ├── If(predicate)           # 条件分支
  ├── IfLastStep(mapping)     # 按最后步骤类型路由
  ├── ObservationControlNode  # 按观察类型路由
  └── GoTo(next_node)         # 无条件跳转
```

## 值得 DMR 借鉴的三个方向

---

### 方向 1：允许受控的 Subagent 嵌套

**问题**：当前 `nested subagent not allowed` 是一个硬限制。但真实场景中经常需要多层委托：
- Agent A 负责"部署应用" → 调用 Agent B "构建镜像" → B 调用 Agent C "运行单元测试"
- Agent A 负责"调查问题" → 调用 Agent B "收集日志" → B 调用 Agent C "分析错误模式"

**TapeAgents 怎么做**：TapeViewStack 允许任意深度嵌套，每层都有自己的 view。

**DMR 应该怎么做**：不需要 TapeViewStack（DMR 用物理 tape 隔离），但应该放开嵌套限制，加深度控制：

```go
// pkg/agent/subagent.go

const (
    subagentMaxSteps = 12
    subagentMaxDepth = 3   // 新增：最大嵌套深度
)

func (a *Agent) RunSubagent(ctx context.Context, parentTape, prompt, modelName, session string) (string, error) {
    // 替换旧的硬编码拒绝逻辑
    depth := countSubagentDepth(parentTape)
    if depth >= subagentMaxDepth {
        return "", fmt.Errorf("subagent: max nesting depth %d reached", subagentMaxDepth)
    }
    
    // 嵌套时递减 maxSteps 防止成本爆炸
    maxSteps := subagentMaxSteps
    if depth > 0 {
        maxSteps = max(4, subagentMaxSteps/(depth+1))
    }
    
    // ... 其余逻辑不变，但 mode.excludeToolNames 不再排除 "subagent"
    mode := &runMode{
        tapeContextOverride: tc,
        maxSteps:            maxSteps,
        // 不再排除 subagent 工具
    }
    // ...
}

func countSubagentDepth(tape string) int {
    return strings.Count(tape, ":"+SubagentTapeSuffix+":")
}
```

**OPA 策略配合**：

```rego
# 用 OPA 控制嵌套深度而不是硬编码
deny_subagent {
    input.tool_name == "subagent"
    count(split(input.tape_name, ":subagent:")) > 3
}
```

**价值**：
- 复杂 DevOps 任务的自然分解（构建→测试→部署的每一步都可以是独立 subagent）
- 保持安全性：通过 OPA 策略 + 深度限制 + 递减 maxSteps 控制成本爆炸
- 向后兼容：默认 `maxDepth=3`，现有行为不变（只是从禁止变成允许 1 层到 2 层嵌套）

---

### 方向 2：声明式编排模式（Orchestration Patterns）

**问题**：当前所有编排逻辑都是 LLM 隐式决定的。LLM 可能：
- 忘记调用某个 subagent
- 以错误的顺序调用
- 不做汇聚就直接输出

对于确定性工作流（CI/CD 管道、审批流程），这种不确定性是不可接受的。

**TapeAgents 怎么做**：Chain 类提供确定性的顺序执行，CallSubagent 节点定义数据流依赖。

**DMR 应该怎么做**：利用已有的 Skill 系统，引入 **Workflow Skill** 概念：

```markdown
<!-- skills/deploy-workflow/SKILL.md -->
---
name: deploy-workflow
description: 标准化部署流程
type: workflow
---

# 部署工作流

## steps

### 1. build
prompt: |
  构建 {{.image}} 镜像，tag 为 {{.tag}}
model: gpt-4o-mini

### 2. test
prompt: |
  对镜像 {{.image}}:{{.tag}} 运行集成测试
depends_on: [build]

### 3. deploy
prompt: |  
  将 {{.image}}:{{.tag}} 部署到 {{.env}} 环境
depends_on: [test]
require_approval: true
```

解析和执行：

```go
// plugins/skill/workflow.go

type WorkflowStep struct {
    Name            string   `yaml:"name"`
    Prompt          string   `yaml:"prompt"`
    Model           string   `yaml:"model,omitempty"`
    DependsOn       []string `yaml:"depends_on,omitempty"`
    RequireApproval bool     `yaml:"require_approval,omitempty"`
}

type Workflow struct {
    Steps []WorkflowStep
}

func (w *Workflow) Execute(ag *agent.Agent, parentTape string, vars map[string]string) (map[string]string, error) {
    results := map[string]string{}
    
    for _, step := range w.TopologicalOrder() {
        // 等待依赖完成
        for _, dep := range step.DependsOn {
            if _, ok := results[dep]; !ok {
                return nil, fmt.Errorf("dependency %s not completed", dep)
            }
        }
        
        // 模板渲染（注入变量 + 前置步骤结果）
        prompt := renderPrompt(step.Prompt, vars, results)
        
        // 作为 subagent 执行
        output, err := ag.RunSubagent(ctx, parentTape, prompt, step.Model, "temp")
        if err != nil {
            return results, fmt.Errorf("step %s failed: %w", step.Name, err)
        }
        results[step.Name] = output
    }
    return results, nil
}
```

**价值**：
- 确定性工作流：步骤顺序和依赖关系是声明式的，不依赖 LLM 决策
- 复用 Skill 生态：Workflow 就是一种特殊的 Skill，可以通过 ClawHub 分发
- 复用 subagent：每个步骤就是一个 subagent 调用，天然继承 OPA 策略和审计
- `require_approval` 与现有审批机制集成

---

### 方向 3：Subagent 间数据传递

**问题**：当前两个 subagent 之间不能直接传递数据。如果步骤 A 的输出需要传给步骤 B，只能：
1. A 的文本结果返回给 parent
2. Parent 的 LLM 理解 A 的结果
3. Parent 把 A 的结果塞进 B 的 prompt

这个过程依赖 LLM 的理解和转述，可能丢失信息（尤其是结构化数据如 JSON、代码块）。

**TapeAgents 怎么做**：
- `CallSubagent(inputs=(-1,))` 声明"输入来自前一个子 agent 的输出"
- `Respond(copy_output=True)` 将子 agent 的最后一步直接拷贝到父 agent 的 view
- `TapeViewStack.outputs_by_subagent` 存储每个子 agent 的输出步骤

**DMR 应该怎么做**：在 subagent 工具上新增 `context` 参数：

```go
// 修改 subagent 工具定义
"properties": map[string]any{
    "prompt":  map[string]any{"type": "string", ...},
    "model":   map[string]any{"type": "string", ...},
    "session": map[string]any{"type": "string", ...},
    // 新增
    "context": map[string]any{
        "type":        "string",
        "description": "Additional context data to prepend to the sub-agent's conversation (e.g., output from a previous subagent). This is injected as a system-level context, not as a user message.",
    },
},
```

执行时注入到子 agent 的 tape：

```go
func (a *Agent) RunSubagent(ctx context.Context, parentTape, prompt, modelName, session, extraContext string) (string, error) {
    // ...
    a.Handoff(childTape, jobID, state)
    
    // 如果有额外上下文，作为 system 消息写入子 tape
    if extraContext != "" {
        contextEntry := tape.NewSystemEntry(fmt.Sprintf(
            "[Context from parent task]\n%s", extraContext,
        ))
        a.tape.Store.Append(childTape, contextEntry)
    }
    
    // ...
}
```

**LLM 使用方式**：
```json
[
  {"name": "subagent", "arguments": {"prompt": "分析日志", "context": ""}},
  {"name": "subagent", "arguments": {
    "prompt": "根据分析结果生成报告",
    "context": "{{上一个 subagent 的输出}}"
  }}
]
```

**价值**：
- LLM 可以显式传递前一个 subagent 的输出给下一个
- 结构化数据（JSON、代码）不经过 LLM 转述，减少信息丢失
- 与 Workflow 模式配合：workflow 引擎自动注入 `context`

---

## 三个方向的关系

```
方向 1（受控嵌套）  ← 基础能力，解锁多层委托
      ↓
方向 3（数据传递）  ← 增强 subagent 间的信息流
      ↓
方向 2（声明式编排）← 最高层抽象，复用 1 和 3 构建确定性工作流
```

建议实施顺序：1 → 3 → 2

## 不值得借鉴的部分

| TapeAgents 能力 | 不采纳理由 |
|-----------------|----------|
| **TapeViewStack（虚拟视图栈）** | DMR 的物理 tape 隔离更适合生产环境（进程安全、持久化、独立审计） |
| **BroadcastLastMessageNode** | DMR 的并行 subagent fan-out 已经覆盖了"广播"场景，且更高效（真并发） |
| **SelectAndCallNode（LLM 选择 agent）** | DMR 中 LLM 通过选择 tool 已经隐式完成了 agent 选择 |
| **Node 图状态机** | DMR 的单循环 + 工具调用模型更简单，对于 DevOps 场景足够；Workflow Skill 覆盖了确定性流程的需求 |
| **`Thought` 类型步骤** | DMR 的 LLM 推理（`<think>` 标签）已经隐式支持，不需要框架级的 Thought 抽象 |

## 代价评估

| 方向 | 改动量 | 风险 |
|------|--------|------|
| 受控嵌套 | ~30 行改 `subagent.go` + OPA 策略 | 低：深度限制 + 递减 maxSteps 控制成本 |
| 数据传递 | ~20 行改 `subagent.go` + 工具定义 | 低：纯增量，不影响现有调用 |
| 声明式编排 | ~300 行新增 `workflow.go` + Skill 解析 | 中：新概念，需要设计 YAML schema |

## 参考

- TapeAgents `Chain`：`tapeagents/team.py:333-342`
- TapeAgents `CallSubagent`：`tapeagents/nodes.py:687-706`
- TapeAgents `TeamAgent.create_team_manager`：`tapeagents/team.py:81-104`
- TapeAgents `TapeViewStack`：`tapeagents/view.py:90-211`
- TapeAgents 控制流节点：`tapeagents/nodes.py:550-684`
- DMR `RunSubagent`：`pkg/agent/subagent.go`
- DMR 并行 fan-out：`pkg/tool/executor.go:108-126, 192-303`
- DMR Skill 系统：`plugins/skill/`
