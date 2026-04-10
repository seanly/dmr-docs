# P1: 三重 Compact 缺乏协调 — 上下文过度丢失

> 灵感来源：TapeAgents `StandardNode.trim_obs_except_last_n` 的渐进式策略

## 问题

DMR 有三套独立的上下文压缩机制，**彼此不协调**：

| 机制 | 触发条件 | 位置 |
|------|---------|------|
| 预防性 compact | token 估算超阈值 | `loop.go:160-197` |
| 工具后 compact | `prompt_tokens >= max_token * threshold` | `loop.go:351-398` |
| 反应式 compact | API 返回 overflow 错误 | `loop.go:200-225` |

**问题场景**：

一个 step 内可能发生：
1. 预防性 compact 触发（估算 token 超了）→ 创建 anchor + summary
2. LLM 调用成功
3. 工具执行完毕
4. 工具后 compact 再次触发（因为 usage 统计显示 token 仍然高）→ **又一次** compact
5. 如果第二次 compact 的 summary 仍然大，下一个 step 的预防性 compact **第三次**触发

每次 compact 都丢弃 anchor 之前的完整上下文，只保留一个 LLM 生成的摘要。三次连续 compact = 摘要的摘要的摘要，**上下文信息严重丢失**。

预防性 compact 有 3-step 冷却（`lastCompactStep`），但与工具后 compact 和反应式 compact 使用**不同的冷却计数器**。没有全局协调。

## TapeAgents 的渐进式策略

TapeAgents 的 `StandardNode` 不做 compact，而是分级裁剪：

```python
# 第 1 级：旧的观察用 short_view()（100 字符摘要）
if obs_index < n_short:
    content = step.short_view()

# 第 2 级：如果仍超 token，把保留完整的观察从 2 条减到 1 条
if count_tokens(messages) > context_size - 500:
    self.trim_obs_except_last_n = 1
```

关键思路：**先轻后重，逐级降级**，而不是一步到位做昂贵的 LLM 摘要。

## 修复方案

### 1. 统一 compact 协调器

```go
// pkg/agent/compact_coordinator.go

type CompactCoordinator struct {
    lastCompactStep int
    cooldown        int  // 全局冷却步数（默认 3）
    compactCount    int  // 当前 anchor 内的 compact 次数
    maxCompacts     int  // 同一 anchor 内最多 compact 几次（默认 2）
}

func (c *CompactCoordinator) ShouldCompact(currentStep int) bool {
    if currentStep - c.lastCompactStep < c.cooldown {
        return false  // 冷却中
    }
    if c.compactCount >= c.maxCompacts {
        return false  // 达到上限，强制进入下一个策略
    }
    return true
}

func (c *CompactCoordinator) RecordCompact(step int) {
    c.lastCompactStep = step
    c.compactCount++
}

func (c *CompactCoordinator) Reset() {
    c.compactCount = 0  // 新 anchor 后重置
}
```

### 2. compact 前先尝试轻量级裁剪

借鉴 TapeAgents 的思路，在触发昂贵的 LLM compact 之前，先尝试裁剪旧的 tool_result：

```go
func (a *Agent) tryLightweightTrim(tapeName string) bool {
    msgs, _ := a.tape.ReadMessages(tapeName, ctx)
    
    // 统计 tool_result 消息
    toolResults := filterByRole(msgs, "tool")
    if len(toolResults) <= 3 {
        return false  // 太少，裁剪无意义
    }
    
    // 对较早的 tool_result 做更激进截断（与 03-graduated-trimming 联动）
    // 如果裁剪后 token 估算低于阈值，跳过 compact
    trimmed := applyGraduatedTrimming(msgs)
    if estimateTokens(trimmed) < threshold {
        return true  // 轻量裁剪足够，不需要 compact
    }
    return false
}
```

### 3. 在 loop.go 中统一使用协调器

```go
// loop.go run() 方法内
coordinator := &CompactCoordinator{cooldown: 3, maxCompacts: 2}

// 替换三处独立的 compact 逻辑：
if needsCompact && coordinator.ShouldCompact(step) {
    // 先尝试轻量裁剪
    if !a.tryLightweightTrim(tapeName) {
        // 轻量裁剪不够，做真正的 compact
        a.compact(tapeName)
        coordinator.RecordCompact(step)
    }
}
```

## 效果

**修复前**（最坏情况）：一个 20-step 任务可能产生 6 次 compact，每次丢失大量上下文

**修复后**：
- 全局冷却 + 上限控制 compact 次数不超过 2 次
- 轻量裁剪优先，很多场景不需要触发 compact
- 上下文保留更完整，agent 表现更好

## 参考

- TapeAgents 渐进裁剪：`tapeagents/nodes.py:204-254`
- DMR 预防性 compact：`pkg/agent/loop.go:160-197`
- DMR 工具后 compact：`pkg/agent/loop.go:351-398`
- DMR 反应式 compact：`pkg/agent/loop.go:200-225`
