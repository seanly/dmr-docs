# P0: 持久记忆系统 — Agent 跨会话记忆与用户建模

> 灵感来源：Hermes Agent `MemoryManager` + `MEMORY.md/USER.md` + `Honcho` 辩证用户建模
>
> DMR 当前每次会话都从零开始理解用户，没有跨会话记忆能力。这是从"工具"进化为"个人助手"的关键缺失。

## 问题

DMR 当前的状态管理完全依赖 Tape。Tape 是优秀的审计日志，但**不是记忆系统**：

| 问题 | 影响 |
|------|------|
| **无跨会话记忆** | 用户每次都要重复说明偏好、环境特征、项目上下文 |
| **无用户画像** | Agent 无法根据用户技能水平调整回复深度 |
| **上下文注入缺失** | system prompt 无法包含历史积累的知识 |
| **Tape 搜索 ≠ 记忆** | `tapeSearch` 是全文检索，不是结构化的知识存储 |

Hermes Agent 通过 `MEMORY.md`（Agent 笔记）+ `USER.md`（用户画像）+ Honcho（辩证推理）三层记忆系统，实现了"越用越懂你"的体验。

## Hermes Agent 的做法

### 1. 双文件记忆存储

```python
# hermes_agent/tools/memory_tool.py:100-405

# MEMORY.md — Agent 的个人笔记
# 内容：环境事实、项目惯例、工具使用技巧
# 上限：2200 字符
# 条目分隔符：\n§\n

# USER.md — 用户画像
# 内容：用户偏好、沟通风格、技能水平、期望
# 上限：1375 字符
```

### 2. 冻结快照 + 实时读取

```python
# memory_tool.py:119-135

# 加载时：capture full state → _system_prompt_snapshot dict
# System prompt 注入：使用冻结快照（整个会话不变 → 保持 prompt cache）
# 工具响应：显示实时状态（反映会话中的写入）
```

这个设计的精妙之处：**system prompt 中的记忆不变 → Anthropic prompt cache 命中率最大化 → 成本降低 90%**。

### 3. 记忆生命周期

```python
# memory_manager.py:167-195

# 每轮对话：
# 1. prefetch_all() → 读取 MEMORY.md + USER.md
# 2. 包裹 <memory-context> 标签 → 注入 system prompt
# 3. LLM 回复
# 4. sync_turn() → 后台更新记忆文件
# 5. queue_prefetch_all() → 预取下轮记忆
```

### 4. 安全扫描

```python
# memory_tool.py:60-98

# 阻止提示注入：
_PROMPT_INJECTION_PATTERNS = [
    r"ignore previous instructions",
    r"act as if you have no",
    r"system prompt override",
]

# 阻止凭据泄露：
_EXFIL_PATTERNS = [
    r"curl.*\$.*KEY",
    r"cat.*\.env",
]
```

### 5. 自动记忆复查

```python
# run_agent.py:2011-2077

# 每 10 轮工具调用触发后台复查线程
# Fork 独立 AIAgent，分析对话历史
# 自动发现值得记住的用户偏好/环境事实
# 静默写入 MEMORY.md / USER.md
```

## DMR 的实现方案

### 整体架构：memory 插件

整个系统实现为**一个 DMR 插件**，零侵入核心代码。

```
plugins/memory/
    memory.go         // 插件入口、Init、RegisterHooks、Shutdown
    store.go          // 文件级记忆存储（MEMORY.md / USER.md）
    tools.go          // memoryRead / memoryWrite / memorySearch 工具定义
    prompt.go         // system prompt 注入逻辑
    security.go       // 内容安全扫描
```

### 1. 插件骨架

```go
// plugins/memory/memory.go

package memory

type MemoryPlugin struct {
    store    *MemoryStore
    snapshot *PromptSnapshot  // 冻结的 system prompt 快照
    config   MemoryConfig
}

func (p *MemoryPlugin) Name() string    { return "memory" }
func (p *MemoryPlugin) Version() string { return "0.1.0" }

func (p *MemoryPlugin) Init(ctx context.Context, config map[string]any) error {
    p.config = parseConfig(config)
    p.store = NewMemoryStore(p.config.MemoryDir)
    p.snapshot = p.store.TakeSnapshot()
    return nil
}

func (p *MemoryPlugin) RegisterHooks(registry *plugin.HookRegistry) {
    // 注入记忆到 system prompt（每轮）
    registry.RegisterHook("SystemPrompt", p.Name(), 50, p.systemPrompt)
    // 注册工具
    registry.RegisterCoreTools(p.Name(), 0, p.registerTools)
    // 每轮结束后同步记忆
    registry.RegisterHook("AfterAgentRun", p.Name(), 30, p.afterRun)
}
```

### 2. 记忆存储

```go
// plugins/memory/store.go

const (
    entryDelimiter = "\n§\n"
    defaultMemoryLimit = 2200  // chars
    defaultUserLimit   = 1375  // chars
)

type MemoryStore struct {
    dir          string        // ~/.dmr/memories/
    mu           sync.RWMutex
    memoryLimit  int
    userLimit    int
}

type MemoryEntry struct {
    Content   string
    CreatedAt string
}

// Read 读取记忆文件，按 § 分隔，自动去重
func (s *MemoryStore) Read(filename string) ([]string, error) {
    s.mu.RLock()
    defer s.mu.RUnlock()

    data, err := os.ReadFile(filepath.Join(s.dir, filename))
    if err != nil {
        if os.IsNotExist(err) {
            return nil, nil
        }
        return nil, err
    }

    entries := strings.Split(string(data), entryDelimiter)
    return deduplicate(entries), nil
}

// Add 添加记忆条目，检查字符上限和重复
func (s *MemoryStore) Add(filename, content string) error {
    s.mu.Lock()
    defer s.mu.Unlock()

    entries, _ := s.readLocked(filename)

    // 检查重复
    for _, e := range entries {
        if strings.TrimSpace(e) == strings.TrimSpace(content) {
            return fmt.Errorf("duplicate entry already exists")
        }
    }

    // 检查字符上限
    limit := s.limitFor(filename)
    currentLen := s.totalLen(entries)
    if currentLen+len(content)+len(entryDelimiter) > limit {
        return fmt.Errorf("would exceed %d char limit (%d/%d used)",
            limit, currentLen, limit)
    }

    entries = append(entries, content)
    return s.writeLocked(filename, entries)
}

// Replace 子串匹配替换
func (s *MemoryStore) Replace(filename, match, replacement string) error {
    s.mu.Lock()
    defer s.mu.Unlock()

    entries, _ := s.readLocked(filename)
    var found []int
    for i, e := range entries {
        if strings.Contains(e, match) {
            found = append(found, i)
        }
    }

    if len(found) == 0 {
        return fmt.Errorf("no entry matching %q", match)
    }
    if len(found) > 1 {
        return fmt.Errorf("ambiguous: %d entries match %q", len(found), match)
    }

    entries[found[0]] = replacement
    return s.writeLocked(filename, entries)
}

// writeLocked 原子写入（临时文件 + rename）
func (s *MemoryStore) writeLocked(filename string, entries []string) error {
    path := filepath.Join(s.dir, filename)
    tmp := path + ".tmp"
    data := strings.Join(entries, entryDelimiter)
    if err := os.WriteFile(tmp, []byte(data), 0600); err != nil {
        return err
    }
    return os.Rename(tmp, path)
}

// TakeSnapshot 冻结当前状态（用于 system prompt 注入）
func (s *MemoryStore) TakeSnapshot() *PromptSnapshot {
    s.mu.RLock()
    defer s.mu.RUnlock()
    memory, _ := s.readLocked("MEMORY.md")
    user, _ := s.readLocked("USER.md")
    return &PromptSnapshot{
        Memory:    memory,
        User:      user,
        TakenAt:   time.Now(),
    }
}
```

### 3. System Prompt 注入

```go
// plugins/memory/prompt.go

type PromptSnapshot struct {
    Memory  []string
    User    []string
    TakenAt time.Time
}

func (p *MemoryPlugin) systemPrompt(ctx context.Context, args ...any) (any, error) {
    snap := p.snapshot  // 使用冻结快照，保持 prompt cache 一致性

    var sb strings.Builder

    if len(snap.Memory) > 0 {
        memText := strings.Join(snap.Memory, entryDelimiter)
        limit := p.config.MemoryLimit
        pct := len(memText) * 100 / limit
        fmt.Fprintf(&sb, "═══════════════════════════════════════════════════\n")
        fmt.Fprintf(&sb, "MEMORY (your personal notes) [%d%% — %d/%d chars]\n",
            pct, len(memText), limit)
        fmt.Fprintf(&sb, "═══════════════════════════════════════════════════\n")
        sb.WriteString(memText)
        sb.WriteString("\n\n")
    }

    if len(snap.User) > 0 {
        userText := strings.Join(snap.User, entryDelimiter)
        limit := p.config.UserLimit
        pct := len(userText) * 100 / limit
        fmt.Fprintf(&sb, "═══════════════════════════════════════════════════\n")
        fmt.Fprintf(&sb, "USER PROFILE [%d%% — %d/%d chars]\n",
            pct, len(userText), limit)
        fmt.Fprintf(&sb, "═══════════════════════════════════════════════════\n")
        sb.WriteString(userText)
    }

    return sb.String(), nil
}
```

### 4. 工具定义

```go
// plugins/memory/tools.go

func (p *MemoryPlugin) registerTools(ctx context.Context, args ...any) (any, error) {
    return []*tool.Tool{
        {
            Name:        "memoryWrite",
            Description: "Write to persistent memory (MEMORY.md for facts, USER.md for user profile). Actions: add, replace, remove.",
            Parameters: map[string]any{
                "type": "object",
                "properties": map[string]any{
                    "target": map[string]any{
                        "type": "string",
                        "enum": []string{"memory", "user"},
                        "description": "memory = agent notes (env facts, conventions); user = user profile (preferences, skills)",
                    },
                    "action": map[string]any{
                        "type": "string",
                        "enum": []string{"add", "replace", "remove"},
                    },
                    "content": map[string]any{
                        "type":        "string",
                        "description": "Content to add, or new content for replace",
                    },
                    "match": map[string]any{
                        "type":        "string",
                        "description": "Substring to match for replace/remove",
                    },
                },
                "required": []string{"target", "action"},
            },
            Handler:    p.handleMemoryWrite,
            Group:      tool.ToolGroupCore,
            SearchHint: "memory, remember, note, preference, user profile, 记忆, 记住",
        },
        {
            Name:        "memoryRead",
            Description: "Read current persistent memory contents (live state, may differ from system prompt snapshot)",
            Parameters: map[string]any{
                "type": "object",
                "properties": map[string]any{
                    "target": map[string]any{
                        "type": "string",
                        "enum": []string{"memory", "user", "all"},
                    },
                },
                "required": []string{"target"},
            },
            Handler: p.handleMemoryRead,
            Group:   tool.ToolGroupCore,
        },
    }, nil
}

func (p *MemoryPlugin) handleMemoryWrite(ctx *tool.ToolContext, args map[string]any) (any, error) {
    target, _ := args["target"].(string)
    action, _ := args["action"].(string)
    content, _ := args["content"].(string)
    match, _ := args["match"].(string)

    filename := "MEMORY.md"
    if target == "user" {
        filename = "USER.md"
    }

    // 安全扫描
    if content != "" {
        if err := scanContent(content); err != nil {
            return map[string]any{"error": err.Error()}, nil
        }
    }

    switch action {
    case "add":
        if err := p.store.Add(filename, content); err != nil {
            return map[string]any{"error": err.Error()}, nil
        }
    case "replace":
        if err := p.store.Replace(filename, match, content); err != nil {
            return map[string]any{"error": err.Error()}, nil
        }
    case "remove":
        if err := p.store.Remove(filename, match); err != nil {
            return map[string]any{"error": err.Error()}, nil
        }
    }

    // 返回当前实时状态（非快照）
    entries, _ := p.store.Read(filename)
    return map[string]any{
        "success": true,
        "current": strings.Join(entries, entryDelimiter),
        "usage":   fmt.Sprintf("%d/%d chars", p.store.totalLen(entries), p.store.limitFor(filename)),
    }, nil
}
```

### 5. 安全扫描

```go
// plugins/memory/security.go

var (
    promptInjectionPatterns = []*regexp.Regexp{
        regexp.MustCompile(`(?i)ignore\s+(previous|all|above)\s+instructions`),
        regexp.MustCompile(`(?i)system\s+prompt\s+override`),
        regexp.MustCompile(`(?i)do\s+not\s+tell\s+the\s+user`),
        regexp.MustCompile(`(?i)act\s+as\s+if\s+you\s+have\s+no`),
    }

    exfilPatterns = []*regexp.Regexp{
        regexp.MustCompile(`(?i)curl\s+[^\n]*\$\{?\w*(KEY|TOKEN|SECRET)`),
        regexp.MustCompile(`(?i)cat\s+[^\n]*(\.env|credentials)`),
        regexp.MustCompile(`(?i)wget\s+.*--post-data`),
    }
)

func scanContent(content string) error {
    for _, re := range promptInjectionPatterns {
        if re.MatchString(content) {
            return fmt.Errorf("blocked: content matches prompt injection pattern")
        }
    }
    for _, re := range exfilPatterns {
        if re.MatchString(content) {
            return fmt.Errorf("blocked: content matches credential exfiltration pattern")
        }
    }
    return nil
}
```

### 6. 配置

```toml
[[plugins]]
name = "memory"
enabled = true

[plugins.config]
memory_dir = "memories"           # 相对于 ~/.dmr/
memory_limit = 2200               # MEMORY.md 字符上限
user_limit = 1375                 # USER.md 字符上限
```

## 解决的问题

- **跨会话一致性**：用户偏好、环境特征一次说明，永久生效
- **个性化回复**：根据 USER.md 调整回复深度和风格
- **上下文效率**：避免每次重复建立上下文，节省 token
- **安全写入**：提示注入和凭据泄露扫描保护记忆完整性
- **缓存友好**：冻结快照设计保持 prompt cache 命中率

## 代价与风险

| 代价 | 评估 |
|------|------|
| **新增插件** | ~500 行 Go，复杂度低 |
| **磁盘占用** | 两个文件共 \<4KB，可忽略 |
| **System prompt 膨胀** | 最多增加 ~3600 chars（两文件上限之和），约 1000 tokens |
| **写入竞争** | 通过 `sync.RWMutex` + 原子写入解决 |
| **记忆过期** | 需要用户或自动复查机制维护（见 #33 后台自学习） |

## 与 #25 自主进化架构的关系

记忆系统是 #25 evolve 插件的**前置依赖**：
- evolve 的模式学习需要记忆系统存储"用户偏好的响应风格"
- evolve 的 Tier 0 精确缓存和记忆系统的"环境事实"互补
- 两者可以共享 `AfterAgentRun` 钩子，但职责不同：记忆系统存知识，evolve 存行为模式

## 参考

- Hermes Agent MemoryManager：`agent/memory_manager.py:72-363`
- Hermes Agent MemoryStore：`tools/memory_tool.py:100-405`
- Hermes Agent 安全扫描：`tools/memory_tool.py:60-98`
- Hermes Agent 冻结快照：`tools/memory_tool.py:119-135`
- Hermes Agent 记忆复查：`run_agent.py:2011-2077`
- DMR SystemPrompt Hook：`pkg/plugin/hooks.go:16`
- DMR AfterAgentRun Hook：`pkg/agent/loop.go:39-41`
- DMR 插件接口：`pkg/plugin/plugin.go:8-14`
