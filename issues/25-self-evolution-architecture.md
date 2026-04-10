# P0: DMR 自主进化架构 — 分层智能减少 LLM 依赖

> 灵感来源：TapeAgents `CachedLLM` + `ReplayLLM` + `Agent.reuse()` + `TrainableLLM` + 微调管线
>
> 这是一个**架构级**设计，不是单一 bug 修复。它定义了 DMR 从"每次调 LLM"进化为"自主处理常见场景"的完整路径。

## 问题

DMR 当前每次交互都依赖远程 LLM。作为个人全能助手长期使用时：

| 问题 | 影响 |
|------|------|
| **成本高** | API 按量计费，重复查询浪费 token |
| **隐私泄露** | 所有对话（含个人信息）发往云端 |
| **速度慢** | 网络往返延迟 1-3 秒，常见操作体验差 |
| **无学习** | 相同问题问 100 次，每次都从零推理 |

## 核心设计：三层智能

类比人类大脑的 System 1（快速直觉）/ System 2（慢速推理）：

```
                       用户输入
                          │
                          ▼
               ┌────────────────────┐
               │  InterceptInput    │ ← evolve 插件 (priority 200)
               │  置信度路由器       │
               └────────────────────┘
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
       ┌──────────┐ ┌──────────┐ ┌──────────┐
       │ Tier 0   │ │ Tier 1   │ │ Tier 2   │
       │ 确定性规则│ │ 本地模型  │ │ 远程 LLM │
       │ SQLite   │ │ ollama   │ │ (不变)    │
       │ <5ms     │ │ 7B-14B   │ │ 完整能力  │
       │ 零成本   │ │ ~100ms   │ │ 当前行为  │
       └──────────┘ └──────────┘ └──────────┘
              │           │           │
              └───────────┼───────────┘
                          ▼
               ┌────────────────────┐
               │  AfterAgentRun     │ ← 学习钩子
               │  模式提取器         │
               └────────────────────┘
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
       ┌──────────┐ ┌──────────┐ ┌──────────┐
       │ 精确缓存 │ │ 模板模式 │ │ 训练数据 │
       │ patterns │ │ patterns │ │ JSONL    │
       │   .db    │ │   .db    │ │ export   │
       └──────────┘ └──────────┘ └──────────┘
```

**核心保证**：系统永远不会比当前行为更差——任何不确定的请求都 fallback 到 Tier 2（原有 LLM 路径，完全不变）。

## 实现方式：evolve 插件

整个系统实现为 **一个 DMR 插件**，零侵入核心代码。

### 注入点

DMR 已有两个完美的注入点：

1. **`InterceptInput` hook**（`pkg/agent/loop.go:69-82`）：
   ```go
   // 已有代码，不需要修改
   if a.plugins != nil && a.plugins.Hooks().HasHook("InterceptInput") {
       result, err := a.plugins.Hooks().CallFirstResult(ctx, "InterceptInput",
           tapeName, prompt, a.config.Workspace, a.tape.Store, a.tape, a)
       if ir, ok := result.(*plugin.InterceptResult); ok && ir != nil {
           return &Result{Output: ir.Output, Steps: 0}, nil  // 完全跳过 LLM
       }
   }
   ```
   evolve 插件以 priority 200（高于 command 插件的 100）注册，对每个用户输入先做路由判断。

2. **`AfterAgentRun` hook**（`pkg/agent/loop.go:39-41`）：
   ```go
   // 已有代码，不需要修改
   if a.plugins != nil && a.plugins.Hooks().HasHook("AfterAgentRun") {
       _, _ = a.plugins.Hooks().CallAll(ctx, "AfterAgentRun", tapeName)
   }
   ```
   每次 LLM 成功执行后触发，evolve 插件提取模式并学习。

### 参考实现

`plugins/command/command.go` 已经是一个完整的 `InterceptInput` 插件参考——处理 `,model.switch` 等命令时直接返回 `InterceptResult` 跳过 LLM。evolve 插件遵循完全相同的模式。

### 插件结构

```
plugins/evolve/
    evolve.go          // 插件入口、Init、RegisterHooks、Shutdown
    router.go          // 置信度路由器：Tier0 → Tier1 → Tier2
    tier0.go           // 确定性规则引擎（精确缓存 + 模板匹配）
    tier1.go           // 本地模型客户端（ollama OpenAI 兼容 API）
    patterns.go        // 模式提取与存储
    training.go        // JSONL 导出（tape → 训练数据）
    feedback.go        // 自修正：跟踪失败、失效模式
    schema.go          // SQLite schema 定义
```

### 插件骨架

```go
package evolve

type EvolvePlugin struct {
    db        *sql.DB          // patterns.db
    tapeStore tape.TapeStore   // 引用 DMR 的 tape store
    tier0     *Tier0Engine
    tier1     *Tier1Client
    router    *Router
    learner   *PatternLearner
    config    EvolveConfig
}

func (p *EvolvePlugin) Name() string { return "evolve" }

func (p *EvolvePlugin) RegisterHooks(registry *plugin.HookRegistry) {
    // 拦截输入：比 command 插件优先级更高
    registry.RegisterHook("InterceptInput", p.Name(), 200, p.intercept)
    // 学习钩子：每次 LLM 执行成功后
    registry.RegisterHook("AfterAgentRun", p.Name(), 50, p.afterRun)
    // 注册工具：,evolve.stats 等
    registry.RegisterCoreTools(p.Name(), 0, p.registerTools)
}

func (p *EvolvePlugin) intercept(ctx context.Context, args ...any) (any, error) {
    tapeName, _ := args[0].(string)
    prompt, _ := args[1].(string)

    // 跳过命令（让 command 插件处理）
    if strings.HasPrefix(strings.TrimSpace(prompt), ",") {
        return nil, nil
    }

    decision := p.router.Route(ctx, tapeName, prompt)
    if decision.Result != nil {
        return decision.Result, nil  // Tier 0 或 Tier 1 命中
    }
    return nil, nil  // 走 Tier 2（不变的 LLM 路径）
}

func (p *EvolvePlugin) afterRun(ctx context.Context, args ...any) (any, error) {
    tapeName, _ := args[0].(string)
    // 后台异步提取模式，不阻塞主流程
    go p.learner.LearnFromLatestExchange(tapeName)
    return nil, nil
}
```

## Tier 0：确定性规则引擎

**目标**：对完全重复或参数化重复的请求，<5ms 零成本响应。

### SQLite Schema (`~/.dmr/var/lib/evolve/patterns.db`)

```sql
-- 精确缓存：MD5(标准化输入) → 响应
CREATE TABLE exact_cache (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    input_hash  TEXT NOT NULL UNIQUE,       -- MD5 of normalized input
    input_norm  TEXT NOT NULL,              -- 标准化后的输入（供调试）
    response    TEXT NOT NULL,              -- 缓存的响应
    tool_calls  TEXT DEFAULT '[]',          -- JSON: 关联的工具调用
    tape_name   TEXT NOT NULL,              -- 来源 tape
    hit_count   INTEGER DEFAULT 0,          -- 命中次数
    miss_count  INTEGER DEFAULT 0,          -- 用户纠正次数
    confidence  REAL DEFAULT 1.0,           -- Wilson 分数下界
    created_at  TEXT NOT NULL,
    last_hit_at TEXT
);

-- 模板模式：参数化规则
CREATE TABLE template_patterns (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    pattern     TEXT NOT NULL,              -- 模板："convert {amount} {from} to {to}"
    action_spec TEXT NOT NULL,              -- JSON: 工具名 + 参数映射
    examples    TEXT DEFAULT '[]',          -- JSON: 具体示例
    hit_count   INTEGER DEFAULT 0,
    miss_count  INTEGER DEFAULT 0,
    confidence  REAL DEFAULT 1.0,
    created_at  TEXT NOT NULL
);

-- 反馈日志
CREATE TABLE pattern_feedback (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    pattern_id   INTEGER NOT NULL,
    pattern_type TEXT NOT NULL,              -- "exact" | "template"
    outcome      TEXT NOT NULL,              -- "hit" | "miss" | "corrected"
    created_at   TEXT NOT NULL
);
```

### 输入标准化

```go
// normalizeInput 生成用于缓存 key 的标准化形式
func normalizeInput(input string) string {
    s := strings.TrimSpace(input)
    s = strings.ToLower(s)
    s = regexp.MustCompile(`\s+`).ReplaceAllString(s, " ")
    s = strings.TrimRight(s, "?!.。？！")
    return s
}

func inputHash(input string) string {
    h := md5.Sum([]byte(normalizeInput(input)))
    return hex.EncodeToString(h[:])
}
```

### 查找逻辑

```go
func (t *Tier0Engine) Lookup(input string) (*Tier0Result, error) {
    hash := inputHash(input)

    // 1. 精确缓存查找
    row := t.db.QueryRow(
        `SELECT response, tool_calls, confidence FROM exact_cache
         WHERE input_hash = ? AND confidence >= ?`, hash, t.minConfidence)
    var resp, toolCalls string
    var conf float64
    if row.Scan(&resp, &toolCalls, &conf) == nil {
        t.db.Exec(`UPDATE exact_cache SET hit_count = hit_count + 1,
                    last_hit_at = ? WHERE input_hash = ?`,
                    time.Now().Format(time.RFC3339), hash)
        return &Tier0Result{Response: resp, Confidence: conf, Source: "exact"}, nil
    }

    // 2. 模板模式匹配
    norm := normalizeInput(input)
    for _, pat := range t.loadActiveTemplates() {
        if match, params := pat.Match(norm); match && pat.Confidence >= t.minConfidence {
            result := pat.Execute(params)
            return &Tier0Result{Response: result, Confidence: pat.Confidence, Source: "template"}, nil
        }
    }

    return nil, nil  // 无命中
}
```

### 学习机制

`AfterAgentRun` 后分析本轮交互：

```go
func (l *PatternLearner) LearnFromLatestExchange(tapeName string) {
    exchange := l.extractLatestExchange(tapeName)
    if exchange == nil || exchange.Status != "ok" {
        return
    }

    // 规则 1：纯文本短响应 → 精确缓存
    if len(exchange.ToolCalls) == 0 && len(exchange.Response) < 500 {
        l.cacheExactResponse(tapeName, exchange.Prompt, exchange.Response)
    }

    // 规则 2：单工具调用 + 参数可从输入确定性导出 → 工具分派规则
    if len(exchange.ToolCalls) == 1 {
        tc := exchange.ToolCalls[0]
        if l.isArgsDeterministicFromInput(exchange.Prompt, tc.Arguments) {
            l.cacheToolDispatch(tapeName, exchange.Prompt, tc, exchange.Response)
        }
    }

    // 规则 3：检查是否有 3+ 个相似请求 → 提取模板
    l.checkForTemplatePattern(tapeName, exchange)
}
```

### 置信度：Wilson 分数下界

```go
// 95% 置信区间的下界，天然处理低样本量下的保守估计
func wilsonScoreLowerBound(hits, misses int) float64 {
    n := float64(hits + misses)
    if n == 0 { return 0.5 }
    p := float64(hits) / n
    z := 1.96 // 95% CI
    denom := 1 + z*z/n
    centre := p + z*z/(2*n)
    spread := z * math.Sqrt(p*(1-p)/n + z*z/(4*n*n))
    return (centre - spread) / denom
}
```

命中阈值 **0.95** — 只有在高度确信时才用缓存。

## Tier 1：本地模型

**目标**：语义相似但非精确匹配的请求，路由到本地 7B-14B 模型。

### 架构

```go
type Tier1Client struct {
    baseURL    string         // http://localhost:11434
    model      string         // qwen2.5:14b
    httpClient *http.Client
    available  atomic.Bool    // 健康检查结果
}

// Chat 调用 ollama 的 OpenAI 兼容 API
func (c *Tier1Client) Chat(ctx context.Context, systemPrompt, prompt string) (string, error) {
    req := map[string]any{
        "model": c.model,
        "messages": []map[string]any{
            {"role": "system", "content": systemPrompt},
            {"role": "user", "content": prompt},
        },
        "stream": false,
    }
    // POST http://localhost:11434/v1/chat/completions
    ...
}
```

### 语义路由

用 `nomic-embed-text`（ollama 嵌入模型，Apple Silicon 上极快）计算输入嵌入，与历史 tape 做余弦相似度匹配：

```go
type EmbeddingIndex struct {
    embeddings [][]float32  // 归一化向量
    prompts    []string     // 对应的原始 prompt
    responses  []string     // 对应的成功响应
}

func (idx *EmbeddingIndex) BestMatch(input string) (similarity float64, matchIdx int) {
    inputEmb := idx.Embed(input)  // 调用 ollama embed API
    best := -1.0
    for i, emb := range idx.embeddings {
        sim := cosineSimilarity(inputEmb, emb)
        if sim > best { best = sim; matchIdx = i }
    }
    return best, matchIdx
}
```

相似度 >= **0.70** → 路由到 Tier 1（配合 few-shot 注入提升质量）。

### Few-shot 注入

Tier 1 调用前，从 tape 中检索 K 个最相似的历史交互作为示例注入 system prompt：

```go
func (c *Tier1Client) BuildPromptWithExamples(basePrompt string, examples []TapeExample) string {
    var sb strings.Builder
    sb.WriteString(basePrompt)
    sb.WriteString("\n\n## 过去的类似交互（参考）:\n\n")
    for _, ex := range examples {
        fmt.Fprintf(&sb, "User: %s\nAssistant: %s\n\n", ex.Prompt, ex.Response)
    }
    return sb.String()
}
```

### 微调管线

```
tape 数据 → JSONL 导出 → unsloth/axolotl SFT 微调 LoRA → ollama 导入 → Tier 1 使用个性化模型
```

```go
// 训练样本格式（兼容 OpenAI fine-tuning API + unsloth/axolotl）
type TrainingSample struct {
    Messages []TrainingMessage `json:"messages"`
}
type TrainingMessage struct {
    Role    string `json:"role"`
    Content string `json:"content"`
}

func ExportTrainingData(store tape.TapeStore) ([]TrainingSample, error) {
    tapes := store.ListTapes()
    var samples []TrainingSample
    for _, name := range tapes {
        entries, _ := store.FetchAll(name, nil)
        for _, exchange := range extractSuccessfulExchanges(entries) {
            samples = append(samples, TrainingSample{
                Messages: []TrainingMessage{
                    {Role: "system", Content: exchange.SystemPrompt},
                    {Role: "user", Content: exchange.Prompt},
                    {Role: "assistant", Content: exchange.Response},
                },
            })
        }
    }
    return samples, nil
}
```

## 置信度路由器

```go
type Router struct {
    tier0  *Tier0Engine
    tier1  *Tier1Client
    index  *EmbeddingIndex
    config RouterConfig
}

func (r *Router) Route(ctx context.Context, tapeName, prompt string) *RouteDecision {
    // TIER 0：确定性规则
    if t0, err := r.tier0.Lookup(prompt); err == nil && t0 != nil {
        return &RouteDecision{Tier: 0, Result: &plugin.InterceptResult{
            Output: t0.Response, Kind: "evolve:tier0",
        }}
    }

    // TIER 1：本地模型（如果可用）
    if r.config.Tier1Enabled && r.tier1.IsAvailable() {
        sim, _ := r.index.BestMatch(prompt)
        if sim >= r.config.Tier1MinConfidence {
            examples := r.index.NearestExamples(prompt, 3)
            resp, err := r.tier1.ChatWithExamples(ctx, prompt, examples)
            if err == nil {
                return &RouteDecision{Tier: 1, Result: &plugin.InterceptResult{
                    Output: resp, Kind: "evolve:tier1",
                }}
            }
            // Tier 1 失败 → 静默降级
        }
    }

    // TIER 2：返回 nil → 走原有 LLM 路径（完全不变）
    return &RouteDecision{Tier: 2, Result: nil}
}
```

## 自修正机制

### 1. 即时纠正检测

```go
func looksLikeCorrection(text string) bool {
    signals := []string{
        "no,", "wrong", "incorrect", "that's not", "actually",
        "try again", "不对", "错了", "不是", "重新", "重来",
    }
    lower := strings.ToLower(text)
    for _, sig := range signals {
        if strings.Contains(lower, sig) { return true }
    }
    return false
}
```

检测到纠正 → 该模式 `miss_count++` → Wilson 分数自动下降 → 低于 0.85 自动禁用。

### 2. 定期审计

后台 goroutine 每日运行：
- 重算所有模式的 Wilson 分数
- 禁用低于阈值的模式
- 清理 30 天未命中的过期模式
- 记录统计到 tape（可审计）

### 3. 用户主动干预

```
,evolve.invalidate <pattern-id>   # 手动失效某个模式
,evolve.reset                     # 清空所有学习数据
,evolve.stats                     # 查看 hit/miss 统计
,evolve.patterns                  # 列出所有活跃模式
```

## 冷启动

| 阶段 | 时间 | 行为 |
|------|------|------|
| **被动观察** | 第 1-7 天 | 全部走 Tier 2（不变）。AfterAgentRun 静默积累。 |
| **首批 Tier 0** | 第 2 周+ | 重复模式出现 3 次后开始拦截最确定的请求。 |
| **启用 Tier 1** | 第 3 周+（用户主动） | 用户配置 ollama → 启用 Tier 1 → 熟悉领域的请求本地处理。 |
| **微调** | 第 4 周+（用户主动） | 积累足够训练数据 → `,evolve.finetune` → 个性化本地模型。 |

已有 tape 历史的用户可以通过 `,evolve.bootstrap` 一次性扫描全部历史，快速提取模式。

## 隐私架构

| 数据 | 存储位置 | 是否出设备 |
|------|----------|-----------|
| patterns.db（所有学习数据） | `~/.dmr/var/lib/evolve/` | **永不** |
| 嵌入向量 | 本地内存 + 磁盘 | **永不** |
| 训练 JSONL | 本地目录 | **永不** |
| Tier 1 推理 | ollama (localhost) | **永不** |
| Tier 2 请求 | 云端 LLM | 是（与当前行为完全相同） |

evolve 插件只**减少**发往云端的数据量，永不增加。

## 配置

```toml
[[plugins]]
name = "evolve"
enabled = true

[plugins.config]
db_path = "var/lib/evolve/patterns.db"

# Tier 0
tier0_enabled = true
tier0_min_confidence = 0.95
tier0_min_occurrences = 3

# Tier 1
tier1_enabled = false          # 启用 ollama 后改为 true
tier1_model = "qwen2.5:14b"
tier1_base_url = "http://localhost:11434"
tier1_min_confidence = 0.70
tier1_embed_model = "nomic-embed-text"

# 训练数据
training_export_dir = "var/lib/evolve/training"

# 自修正
feedback_min_accuracy = 0.85
max_patterns = 10000
```

## 实施路线

### Phase 1：精确缓存（~2 周）
- `plugins/evolve/` 骨架 + InterceptInput + AfterAgentRun
- patterns.db + exact_cache CRUD
- `,evolve.stats` 命令
- 目标：重复问题 <5ms 返回

### Phase 2：模板模式 + 自修正（~2 周）
- 模板提取和匹配引擎
- FeedbackTracker + Wilson 分数 + 审计循环
- `,evolve.patterns` / `,evolve.invalidate`

### Phase 3：本地模型路由（~3 周）
- Tier1Client (ollama API) + 健康检查
- 嵌入索引 + 余弦相似度 + Router 三层路由
- Few-shot 注入

### Phase 4：训练管线（~1 周）
- tape → JSONL 导出
- `,evolve.export` / `,evolve.bootstrap` / `,evolve.finetune`

## TapeAgents 灵感映射

| TapeAgents 机制 | DMR evolve 对应 |
|-----------------|-----------------|
| `CachedLLM` (MD5 hash → JSONL cache) | Tier 0 exact_cache (MD5 hash → SQLite) |
| `ReplayLLM` (精确 prompt 匹配 + Levenshtein 诊断) | Tier 0 template matching (模糊匹配) |
| `Agent.reuse()` (从 tape 重建 prompt-output 对) | `ExportTrainingData()` (从 tape 提取训练样本) |
| `TrainableLLM` (vLLM 服务 + 在线学习) | Tier1Client (ollama 服务 + LoRA 微调) |
| `run_finetuning_loop()` (SFT + GRPO) | `,evolve.finetune` 工作流 (SFT via unsloth) |
| Teacher-student distillation (大模型 tape → 训练小模型) | Tier 2 tape → 训练 Tier 1 本地模型 |

## 参考

- TapeAgents CachedLLM：`tapeagents/llms/cached.py`
- TapeAgents ReplayLLM：`tapeagents/llms/replay.py`
- TapeAgents Agent.reuse()：`tapeagents/agent.py:783-837`
- TapeAgents TrainableLLM：`tapeagents/llms/trainable.py`
- TapeAgents 微调管线：`tapeagents/finetune/`
- DMR InterceptInput：`pkg/agent/loop.go:69-82`
- DMR AfterAgentRun：`pkg/agent/loop.go:39-41`
- DMR InterceptResult：`pkg/plugin/intercept.go`
- DMR command 插件（参考）：`plugins/command/command.go`
- DMR HookRegistry：`pkg/plugin/hooks.go`
- DMR ChatProvider：`pkg/core/execution.go:18-21`
