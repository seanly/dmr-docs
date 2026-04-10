# P2: 分级观察裁剪（Graduated Observation Trimming）

> 灵感来源：TapeAgents `StandardNode.trim_obs_except_last_n` + `short_view()`

## 问题

DMR 有两种上下文管理手段，但中间缺一层：

| 手段 | 成本 | 效果 |
|------|------|------|
| `tool_result_max_chars` 截断 | 零（字符串裁剪） | 对所有 tool_result 统一截断，不区分新旧 |
| `compact`（LLM 摘要） | 高（一次额外 LLM 调用） | 创建 anchor + summary，重置上下文窗口 |

问题在于：当 anchor 内的上下文逐渐膨胀时，所有 tool_result 享受**同等的上下文预算**。一个 10 轮前的 web 搜索结果和最近一轮的文件读取结果，截断长度相同。这浪费了宝贵的上下文空间，也过早触发了昂贵的 compact 操作。

## TapeAgents 的做法

```python
class StandardNode(Node):
    trim_obs_except_last_n: int = 2  # 保留最近 2 条完整

    def steps_to_messages(self, steps):
        n_observations = sum(1 for s in steps if isinstance(s, Observation))
        n_short = n_observations - self.trim_obs_except_last_n
        
        obs_index = 0
        for step in steps:
            if isinstance(step, Observation):
                if obs_index < n_short:
                    content = step.short_view()  # 压缩摘要（默认 100 字符）
                else:
                    content = step.llm_view()    # 完整内容
                obs_index += 1
```

两级显示：远处的观察用 `short_view()` 压缩，近处的保留完整。还有一步降级：如果仍然超 token，把 `trim_obs_except_last_n` 从 2 降到 1。

## DMR 的实现方案

### 在 ReadMessages 中对 tool_result 分级截断

```go
// pkg/tape/manager.go 修改 ReadMessages

type TrimConfig struct {
    RecentToolResults   int  // 最近 N 条保留完整（默认 3）
    OldMaxChars         int  // 较早的 tool_result 截断到此长度（默认 max_chars/4）
}

func (m *TapeManager) ReadMessages(tape string, ctx *TapeContext) ([]map[string]any, error) {
    entries, err := m.Store.FetchAll(tape, opts)
    // ...
    
    // 第一遍：统计 tool_result 总数
    toolResultCount := 0
    for _, e := range entries {
        if e.Kind == "tool_result" { toolResultCount++ }
    }
    
    // 第二遍：构建 messages，对较早的 tool_result 更激进截断
    nShort := toolResultCount - m.trimConfig.RecentToolResults
    toolResultIndex := 0
    
    for _, e := range entries {
        msg := entryToMessage(e)
        if e.Kind == "tool_result" {
            if toolResultIndex < nShort {
                msg = truncateToolResult(msg, m.trimConfig.OldMaxChars)
            }
            // 最近的 tool_result 保持正常的 tool_result_max_chars 截断
            toolResultIndex++
        }
        messages = append(messages, msg)
    }
    
    return messages, nil
}
```

### 截断策略

```go
func truncateToolResult(msg map[string]any, maxChars int) map[string]any {
    content, ok := msg["content"].(string)
    if !ok || len([]rune(content)) <= maxChars {
        return msg
    }
    
    runes := []rune(content)
    head := maxChars * 3 / 4   // 75% 头部
    tail := maxChars - head     // 25% 尾部
    
    truncated := string(runes[:head]) + 
        fmt.Sprintf("\n\n[... truncated %d chars (old result, showing summary) ...]\n\n", 
            len(runes)-head-tail) +
        string(runes[len(runes)-tail:])
    
    msg["content"] = truncated
    return msg
}
```

### 配置

```toml
[agent]
# 现有配置
tool_result_max_chars = 20000

# 新增配置
recent_tool_results = 3        # 最近 3 条保留完整截断长度
old_tool_result_ratio = 0.25   # 较早的截断到 max_chars * 0.25 = 5000
```

## 效果对比示例

假设 anchor 内有 8 轮 tool_result，每条原始 15000 字符，`tool_result_max_chars = 20000`：

**当前（无分级）**：
```
tool_result_1: 15000 chars (完整)
tool_result_2: 15000 chars (完整)
...
tool_result_8: 15000 chars (完整)
总计: ~120000 chars → 容易触发 compact
```

**分级后（recent=3, old_ratio=0.25）**：
```
tool_result_1: 3750 chars (旧，激进截断)
tool_result_2: 3750 chars
tool_result_3: 3750 chars
tool_result_4: 3750 chars
tool_result_5: 3750 chars
tool_result_6: 15000 chars (近，完整)
tool_result_7: 15000 chars
tool_result_8: 15000 chars
总计: ~63750 chars → compact 触发延后，节省一次 LLM 调用
```

## 解决的问题

- **减少 compact 频率**：每次 compact 需要一次额外 LLM 调用，成本和延迟不低
- **小上下文模型友好**：Ollama 本地模型通常只有 4K-8K 上下文，分级裁剪大幅提升可用空间
- **长工具链优化**：web 搜索、文件读取等任务产生大量中间结果，早期结果价值衰减快

## 代价与风险

- 只修改 `ReadMessages`，约 30 行改动
- 风险：过度截断旧 tool_result 可能导致 LLM 丢失关键上下文
  - 缓解：默认 `recent=3` 和 `ratio=0.25` 保守配置
  - 缓解：截断提示文本告知 LLM 内容被压缩
- 不影响 tape 存储（原始数据完整保留，只在读取构建 messages 时截断）

## 可扩展性

高。未来可以增加更细粒度的策略：
- 按 tool_result 的 kind 区分（搜索结果 vs 文件内容 vs 代码输出）
- 按 token 预算动态调整截断级别
- LLM 评估哪些旧结果仍然重要（但这又回到 compact 的成本了）

## 参考

- TapeAgents `StandardNode.steps_to_messages`：`tapeagents/nodes.py:204-254`
- TapeAgents `Observation.short_view()`：`tapeagents/core.py:116-118`
- DMR 现有截断：`pkg/agent/loop.go` `truncateForProvider()`
- DMR compact 机制：`pkg/agent/loop.go` 自动 handoff 逻辑
