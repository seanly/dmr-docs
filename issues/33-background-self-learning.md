# P1: 后台自学习 — AfterAgentRun 驱动的自动知识提取

> 灵感来源：Hermes Agent 后台复查线程 + `_SKILL_REVIEW_PROMPT` + `_MEMORY_REVIEW_PROMPT` + nudge 机制
>
> 这是 #30 记忆系统和 #32 技能自创建的**驱动引擎**。它决定"什么时候学"和"怎么触发学习"。

## 问题

即使有了记忆系统（#30）和技能创建能力（#32），如果只靠 Agent 在对话中"自觉"调用 `memoryWrite` / `skillCreate`，实际效果会很差：

| 问题 | 原因 |
|------|------|
| **Agent 注意力在任务上** | 执行复杂任务时，Agent 专注于完成用户请求，不会主动反思 |
| **记忆写入时机不明确** | Agent 不知道哪些信息值得持久化 |
| **技能创建标准模糊** | Agent 不知道什么任务值得沉淀为技能 |
| **用户体验中断** | 如果在对话中主动复查，会打断用户的工作流 |

需要一个**独立的后台学习机制**，在每次任务完成后静默分析、提取、持久化。

## Hermes Agent 的做法

### 后台复查架构

```python
# hermes_agent/run_agent.py:2011-2077

# 触发条件：
# _iters_since_skill >= 10 (每 10 次工具调用触发一次)
# _turns_since_memory >= 10 (每 10 轮用户对话触发一次)

# 执行方式：
# 1. Fork 独立 AIAgent 线程（相同模型、相同工具）
# 2. 复制当前对话历史
# 3. 追加复查提示（_SKILL_REVIEW_PROMPT / _MEMORY_REVIEW_PROMPT）
# 4. 独立运行（max_iterations=8，无用户可见输出）
# 5. 直接调用 memory_tool() 和 skill_manage() 写入持久化
# 6. stdout/stderr → /dev/null（用户无感）
```

### 复查提示

```python
# run_agent.py:2042-2058 — 技能复查
_SKILL_REVIEW_PROMPT = """
Look at the conversation above.
Is there something worth saving as a reusable skill?

Create when:
- Complex task succeeded (5+ tool calls)
- You overcame errors through trial
- User-corrected approach worked

If yes, call skill_manage(action='create', ...).
If no, respond with 'No skill to create.'
"""

# run_agent.py:~2010 — 记忆复查
_MEMORY_REVIEW_PROMPT = """
Identify user preferences, environment facts,
or project conventions worth remembering.
"""
```

### 计数器重置

```python
# run_agent.py:6603
# 当 skill_manage 被实际调用时，重置计数器
# 避免刚创建完技能又触发复查
```

## DMR 的实现方案

### 整体架构：learner 插件

实现为独立插件，通过 `AfterAgentRun` Hook 触发后台学习。与 #30 记忆插件和 #32 技能插件**解耦**——learner 只负责触发和分析，实际写入由 memory/skill 工具完成。

```
plugins/learner/
    learner.go       // 插件入口
    analyzer.go      // 对话分析逻辑
    prompts.go       // 复查提示模板
```

### 1. 插件骨架

```go
// plugins/learner/learner.go

package learner

type LearnerPlugin struct {
    config       LearnerConfig
    iterCounter  atomic.Int64  // 工具调用计数器
    turnCounter  atomic.Int64  // 用户轮次计数器
    lastLearnAt  atomic.Value  // 上次学习时间（防抖）
    agentRunner  AgentRunner   // 从 HostService 注入
}

type LearnerConfig struct {
    Enabled           bool   `mapstructure:"enabled"`
    SkillNudgeInterval int   `mapstructure:"skill_nudge_interval"` // 默认 10
    MemoryNudgeInterval int  `mapstructure:"memory_nudge_interval"` // 默认 10
    MaxReviewSteps    int    `mapstructure:"max_review_steps"`      // 默认 8
    MinCooldownSec    int    `mapstructure:"min_cooldown_sec"`      // 默认 60
}

func (p *LearnerPlugin) Name() string    { return "learner" }
func (p *LearnerPlugin) Version() string { return "0.1.0" }

func (p *LearnerPlugin) RegisterHooks(registry *plugin.HookRegistry) {
    // 在每次 agent 执行完后触发
    registry.RegisterHook("AfterAgentRun", p.Name(), 20, p.afterRun)
}
```

### 2. 触发逻辑

```go
// plugins/learner/learner.go (续)

func (p *LearnerPlugin) afterRun(ctx context.Context, args ...any) (any, error) {
    tapeName, _ := args[0].(string)

    p.iterCounter.Add(1)

    // 防抖：距上次学习不到 cooldown 秒
    if last, ok := p.lastLearnAt.Load().(time.Time); ok {
        if time.Since(last) < time.Duration(p.config.MinCooldownSec)*time.Second {
            return nil, nil
        }
    }

    needSkillReview := p.iterCounter.Load() >= int64(p.config.SkillNudgeInterval)
    needMemoryReview := p.turnCounter.Load() >= int64(p.config.MemoryNudgeInterval)

    if !needSkillReview && !needMemoryReview {
        return nil, nil
    }

    // 后台异步执行，不阻塞主流程
    go p.runBackgroundReview(tapeName, needSkillReview, needMemoryReview)

    // 重置计数器
    if needSkillReview {
        p.iterCounter.Store(0)
    }
    if needMemoryReview {
        p.turnCounter.Store(0)
    }
    p.lastLearnAt.Store(time.Now())

    return nil, nil
}
```

### 3. 后台复查执行

```go
// plugins/learner/analyzer.go

func (p *LearnerPlugin) runBackgroundReview(tapeName string, skill, memory bool) {
    ctx, cancel := context.WithTimeout(context.Background(), 120*time.Second)
    defer cancel()

    // 构建复查提示
    var prompt strings.Builder
    prompt.WriteString("Review the conversation above and perform the following:\n\n")

    if memory {
        prompt.WriteString(memoryReviewPrompt)
        prompt.WriteString("\n\n")
    }
    if skill {
        prompt.WriteString(skillReviewPrompt)
    }

    // 使用 AgentRunner 在子 tape 中执行复查
    // 子 tape 继承主 tape 的历史（可读），但结果写入独立 tape
    childTape := fmt.Sprintf("%s:learner:%s", tapeName, shortID())

    if p.agentRunner == nil {
        return // 无法运行（可能是测试环境）
    }

    _, err := p.agentRunner.Run(ctx, childTape, prompt.String(), nil)
    if err != nil {
        // 静默失败，不影响主流程
        log.Printf("[learner] background review failed for %s: %v", tapeName, err)
    }
}
```

### 4. 复查提示模板

```go
// plugins/learner/prompts.go

const memoryReviewPrompt = `## Memory Review

Look at the conversation above and identify information worth persisting:

1. **Environment facts**: OS, language versions, tools installed, project structure
2. **User preferences**: communication style, preferred tools, coding conventions
3. **Project conventions**: naming patterns, deployment targets, testing approaches

For each finding, call memoryWrite(target="memory"|"user", action="add", content="...").

Rules:
- Only save non-obvious facts (skip things derivable from code)
- Keep entries concise (one fact per entry)
- Check existing memory first to avoid duplicates
- If nothing worth saving, respond "No new memories to save."`

const skillReviewPrompt = `## Skill Review

Look at the conversation above and determine if the task is worth saving as a reusable skill.

Create a skill when ALL of these apply:
- The task required 5+ tool calls to complete
- You overcame errors or discovered non-obvious steps
- The procedure would be useful for similar future tasks

If creating, call skillCreate with:
- Clear trigger conditions (when to use this skill)
- Numbered steps with exact commands
- Pitfalls section (what went wrong, how to avoid)
- Verification steps (how to confirm success)

If the task was too simple or too specific, respond "No skill to create."`
```

### 5. 与 AgentRunner 的集成

learner 插件需要 `AgentRunner` 接口来执行后台 agent。这个接口已在 DMR 中存在（`pkg/plugin/external.go:226-238`）：

```go
// 已有接口，不需修改
// pkg/plugin/external.go:226
type AgentRunner interface {
    Run(ctx context.Context, tapeName, prompt string, pluginCtx json.RawMessage) (*AgentResult, error)
}
```

在 Init 中注入：

```go
func (p *LearnerPlugin) Init(ctx context.Context, config map[string]any) error {
    p.config = parseConfig(config)
    // AgentRunner 通过 config 中的特殊 key 注入
    // 这与 subagent 插件获取 AgentRunner 的方式相同
    if runner, ok := config["_agent_runner"].(AgentRunner); ok {
        p.agentRunner = runner
    }
    return nil
}
```

### 6. 配置

```toml
[[plugins]]
name = "learner"
enabled = true

[plugins.config]
skill_nudge_interval = 10    # 每 10 次工具调用触发技能复查
memory_nudge_interval = 10   # 每 10 轮对话触发记忆复查
max_review_steps = 8         # 后台复查最多 8 步
min_cooldown_sec = 60        # 两次复查间最短间隔
```

## 解决的问题

- **自动知识提取**：无需用户主动要求，Agent 自动从对话中提取有价值的信息
- **无感学习**：后台异步执行，不打断用户工作流
- **适度频率**：计数器 + 防抖机制避免过度学习
- **解耦设计**：learner 只负责触发，实际写入由 memory/skill 工具完成
- **可审计**：复查过程记录在子 tape 中，可追溯

## 代价与风险

| 代价 | 评估 |
|------|------|
| **额外 LLM 调用** | 每次后台复查消耗 1-8 轮 LLM 调用（使用主模型） |
| **成本控制** | 通过 nudge_interval 和 cooldown 限制频率；可配置使用更便宜的辅助模型 |
| **质量风险** | 后台 Agent 可能创建低质量记忆/技能 → 安全扫描 + frontmatter 校验缓解 |
| **竞争条件** | 后台写入与用户主动写入可能冲突 → 文件锁（#30）解决 |

## 与其他 Issue 的关系

```
#33 后台自学习
  ├── 触发 → #30 记忆系统 (memoryWrite)
  ├── 触发 → #32 技能自创建 (skillCreate)
  ├── 被利用 ← #25 自主进化 (evolve 的 afterRun 学习机制覆盖更广)
  └── 互补 ← Tape 审计 (复查结果记录在子 tape 中)
```

**注意**：如果 #25 自主进化架构实现后，learner 的记忆/技能复查功能可以**合并到 evolve 插件的 `afterRun` 逻辑中**，避免两个插件做相似的后台分析。

## 参考

- Hermes Agent 后台复查线程：`run_agent.py:2011-2077`
- Hermes Agent 技能 nudge 触发：`run_agent.py:7735-7737`
- Hermes Agent 记忆 nudge 触发：`run_agent.py:7495-7501`
- Hermes Agent 复查提示：`run_agent.py:2042-2058`
- DMR AfterAgentRun Hook：`pkg/agent/loop.go:39-41`
- DMR AgentRunner 接口：`pkg/plugin/external.go:226-238`
- DMR subagent 实现参考：`plugins/subagent/subagent.go`
