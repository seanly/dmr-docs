# P2: Deny Always — 审批系统持久化拒绝选项

## 问题

当前 DMR 的 CLI 审批系统（`plugins/cli/approver.go`）提供四个选项：

- `y` — 本次允许（ApprovedOnce）
- `s` — 会话内允许（ApprovedSession）
- `a` — 永远允许（ApprovedAlways，持久化到 `approvals.json`）
- `n` — 拒绝

但**没有 "永远拒绝" 选项**。用户每次遇到危险命令（如 `rm -rf /`、`DROP TABLE`）都需要手动拒绝。对于高频使用场景，这是一个安全隐患——一次走神就可能误批。

### 场景

1. LLM 反复尝试 `rm -rf` 清理临时文件 → 用户每次都要拒绝
2. LLM 尝试 `git push --force` → 用户希望永远禁止
3. LLM 尝试 `docker rm -f` 删除容器 → 某些环境下必须禁止

### 现状

```go
// plugins/cli/approver.go — 审批循环
// 只有 y/s/a/n 四个选项
// OPA 的 allow_rules 支持 glob 匹配，但只用于允许，不用于拒绝
```

OPA 策略层可以写 `deny` 规则，但需要用户手动编辑 Rego 文件，门槛高。

## 设计方案

### 1. 新增审批选项

```text
[y]es  [s]ession  [a]lways  [n]o  [d]eny always  [?]help
```

- `d` — DeniedAlways：持久化拒绝，记录到 `denials.json`，后续匹配到相同 pattern 时自动拒绝

### 2. 拒绝规则存储

```go
// plugins/opapolicy/denial_cache.go

type DenialEntry struct {
    Pattern   string    `json:"pattern"`    // 工具名:参数 glob 模式
    Reason    string    `json:"reason"`     // 拒绝原因（可选）
    CreatedAt time.Time `json:"created_at"`
}

// 存储在 ~/.dmr/denials.json
type DenialCache struct {
    entries []DenialEntry
    path    string
}

func (c *DenialCache) IsDenied(toolName string, args map[string]any) bool {
    pattern := buildPattern(toolName, args)
    for _, e := range c.entries {
        if matchGlob(e.Pattern, pattern) {
            return true
        }
    }
    return false
}
```

### 3. Pattern 匹配策略

当用户选择 `d` 时，系统需要决定拒绝的 pattern 粒度：

```text
Deny always? Choose pattern:
  [1] shell:rm -rf /        ← 精确匹配（仅此命令）
  [2] shell:rm -rf *        ← 拒绝所有 rm -rf 开头的命令
  [3] shell:rm *            ← 拒绝所有 rm 命令
  [4] custom pattern        ← 自定义 glob
```

默认选择最具体的模式（选项 1），高级用户可选择更宽泛的模式。

### 4. 与 OPA 策略引擎集成

在 `BatchBeforeToolCall` hook 中，优先检查 denial cache：

```go
// plugins/opapolicy/opapolicy.go

func (p *Plugin) batchBeforeToolCall(ctx context.Context, calls []ToolCallInfo) ([]Decision, error) {
    decisions := make([]Decision, len(calls))
    for i, call := range calls {
        // 优先检查持久化拒绝
        if p.denialCache.IsDenied(call.ToolName, call.Args) {
            decisions[i] = Decision{Action: "deny", Reason: "permanently denied by user"}
            continue
        }
        // 然后走正常 OPA 评估
        decisions[i] = p.evaluate(call)
    }
    return decisions, nil
}
```

### 5. 管理命令

```text
,denials.list               — 列出所有永久拒绝规则
,denials.remove <pattern>   — 移除指定拒绝规则
,denials.clear              — 清空所有拒绝规则
```

## 实现步骤

- [ ] 在 CLI approver 中添加 `d` (deny always) 选项
- [ ] 实现 `DenialCache`（加载/保存 `~/.dmr/denials.json`）
- [ ] 实现 pattern 匹配逻辑（复用 allow_rules 的 glob 匹配）
- [ ] 在 OPA 插件的 `BatchBeforeToolCall` 中集成 denial 检查
- [ ] 添加 `,denials.*` 管理命令
- [ ] Web approver 同步支持 deny always
- [ ] 添加测试

## 代价与风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| 过于宽泛的 deny pattern 导致正常操作被阻断 | 中 | 高 | 默认使用精确模式；提供 `,denials.list` 和 `,denials.remove` |
| denial 规则与 OPA allow 规则冲突 | 低 | 中 | denial 优先级高于 allow（安全优先） |
| denials.json 文件损坏 | 低 | 低 | JSON 解析失败时降级为空列表（不拒绝） |

## 参考

- DMR `plugins/cli/approver.go` — 当前审批实现
- DMR `plugins/opapolicy/allow_rules.go` — allow_rules glob 匹配（可复用）
- DMR `plugins/opapolicy/approval_cache.go` — ApprovedAlways 持久化机制（对称实现）
