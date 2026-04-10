# P1: 集成投票标注 — 用离线投票给 Tape 数据打质量标签

> 灵感来源：TapeAgents GAIA Agent 的 `majority_vote` (maj@3) + #25 自主进化架构的数据质量问题

## 问题

#25 自主进化架构的核心瓶颈不是算法，而是**数据质量**：

1. **Tier 0 缓存**：缓存了错误响应怎么办？用户沉默不代表正确
2. **Tier 1 路由**：嵌入相似度无法判断"这个任务本地模型能不能搞定"
3. **微调数据**：历史 tape 中混杂着正确和错误的交互，直接用于训练会学到错误

根本原因：**没有可靠的质量信号**来区分 tape 中哪些交互是可信的。

## TapeAgents 的启发

GAIA Agent 的集成投票数据（`examples/gaia_agent/README.md`）：

| 模型 | 单次 | maj@3 | 提升 |
|------|------|-------|------|
| GPT-4o mini | 27.3% | 32.3% | +5.0 |
| Sonnet 3.7 | 53.9% | 55.8% | +1.9 |

关键洞察：
- **小模型从投票中获益更大**（错误率高 → 纠错空间大）
- **投票共识度本身就是质量信号**——3/3 一致说明任务确定性高，无共识说明任务难/开放

## 方案：离线投票标注

### 核心思路

对历史 tape 中的每条交互，用本地模型离线跑多次推理，用投票结果作为质量标签。

```
历史 tape 中的每条交互
        │
        ▼
  提取原始 prompt + context
        │
        ▼
  用本地模型跑 k 次（默认 k=3）
        │
        ├── k/k 一致 且 与原始响应一致  → high_confidence
        ├── 多数一致 且 与原始响应一致  → medium_confidence
        ├── k/k 一致 但 与原始响应不同  → original_suspect
        └── 无共识                      → ambiguous
```

### 标签含义与用途

| 标签 | 含义 | 对进化系统的影响 |
|------|------|-----------------|
| `high_confidence` | 确定性任务，小模型也能做对 | 可进入 Tier 0 缓存 + Tier 1 few-shot |
| `medium_confidence` | 有难度但可收敛 | 只作为 Tier 1 few-shot，不缓存 |
| `original_suspect` | 原始 LLM 响应可能有错 | 排除出缓存/训练集，标记待人工审查 |
| `ambiguous` | 开放性/高难度任务 | 永远走 Tier 2，不尝试本地处理 |

### 与 #25 进化架构的集成

**替代嵌入相似度路由**：不再用余弦相似度猜测"能不能走 Tier 1"，而是靠投票实证：

```
Tier 1 路由决策：
  旧方案：embed(input) → cosine_similarity >= 0.70 → 走 Tier 1  ← 不可靠
  新方案：该 task_type 有 high_confidence 标注记录 → 走 Tier 1  ← 实证验证
```

**筛选训练数据**：

```
只有 high_confidence → 进入精确缓存
只有 high/medium    → 作为 few-shot 示例
ambiguous           → 排除
original_suspect    → 排除 + 告警
```

### SQLite Schema

```sql
-- 在 patterns.db 中新增

CREATE TABLE consensus_labels (
    id             INTEGER PRIMARY KEY AUTOINCREMENT,
    tape_name      TEXT NOT NULL,
    entry_index    INTEGER NOT NULL,
    input_hash     TEXT NOT NULL,       -- 原始输入的 MD5
    consensus      REAL NOT NULL,       -- 0.0 ~ 1.0 (一致性比例)
    match_original BOOLEAN NOT NULL,    -- 投票结果是否与原始响应一致
    confidence     TEXT NOT NULL,       -- "high" | "medium" | "suspect" | "ambiguous"
    vote_count     INTEGER NOT NULL,    -- 投票次数 (k)
    vote_hashes    TEXT NOT NULL,       -- JSON: 每次推理结果的摘要 hash
    model_used     TEXT NOT NULL,       -- 投票时使用的模型
    labeled_at     TEXT NOT NULL,
    UNIQUE(tape_name, entry_index)
);

CREATE INDEX idx_consensus_confidence ON consensus_labels(confidence);
CREATE INDEX idx_consensus_input ON consensus_labels(input_hash);
```

### 投票一致性判断

纯文本响应用语义相似度（而非精确匹配），因为同一个正确答案可以有不同措辞：

```go
// plugins/evolve/labeler.go

type VoteResult struct {
    Response string
    Hash     string // SHA256(normalized_response)
}

// 判断两个响应是否"语义一致"
// 策略 1：结构化输出 → 精确比较（工具调用名 + 参数）
// 策略 2：纯文本 → 提取关键事实，比较事实集合
// 策略 3：兜底 → 用 embed 余弦相似度 >= 0.95
func responsesAgree(a, b VoteResult, mode string) bool {
    switch mode {
    case "tool_call":
        // 工具名 + 参数完全一致
        return a.Hash == b.Hash
    case "factual":
        // 提取数字/实体，集合交集 >= 80%
        return factOverlap(a.Response, b.Response) >= 0.8
    default:
        // 嵌入余弦相似度（高阈值，因为这里要求"一致"而非"相似"）
        return cosineSimilarity(embed(a.Response), embed(b.Response)) >= 0.95
    }
}
```

### 与原始响应的比较

```go
// 投票结果 vs 原始 tape 中记录的 LLM 响应
func classifyConsensus(votes []VoteResult, original string, k int) ConsensusLabel {
    // 1. 找多数答案
    majority, majorityCount := findMajority(votes)
    consensus := float64(majorityCount) / float64(k)
    
    // 2. 多数答案是否与原始一致
    matchOriginal := responsesAgree(majority, VoteResult{Response: original}, "auto")
    
    // 3. 分类
    var confidence string
    switch {
    case consensus >= 1.0 && matchOriginal:
        confidence = "high"
    case consensus >= 0.66 && matchOriginal:
        confidence = "medium"
    case consensus >= 1.0 && !matchOriginal:
        confidence = "suspect"  // 投票全票否决原始响应
    default:
        confidence = "ambiguous"
    }
    
    return ConsensusLabel{
        Consensus:     consensus,
        MatchOriginal: matchOriginal,
        Confidence:    confidence,
    }
}
```

### 离线执行

利用 Mac 空闲时间（夜间/低负载时段）后台运行：

```go
// plugins/evolve/labeler.go

type Labeler struct {
    db       *sql.DB
    ollama   *Tier1Client
    model    string
    votes    int           // 默认 3
    interval time.Duration // 标注间隔，避免占满 CPU
}

// 后台 goroutine，在 evolve 插件 Init 时启动
func (l *Labeler) Run(ctx context.Context) {
    ticker := time.NewTicker(l.interval)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            // 找到下一条未标注的交互
            entry, err := l.nextUnlabeled(ctx)
            if err != nil || entry == nil {
                continue
            }
            
            // 跑 k 次投票
            votes, err := l.runVotes(ctx, entry, l.votes)
            if err != nil {
                continue
            }
            
            // 分类并存储
            label := classifyConsensus(votes, entry.OriginalResponse, l.votes)
            l.saveLabel(ctx, entry, label)
        }
    }
}
```

### CLI 命令

```bash
# 手动触发批量标注
dmr evolve label --tape "daily:*" --model qwen2.5:7b --votes 3

# 查看标注统计
dmr evolve label-stats
# Output:
# Total labeled: 847
# high_confidence: 312 (36.8%)
# medium_confidence: 198 (23.4%)
# original_suspect: 45 (5.3%)    ← 这些原始响应可能有错
# ambiguous: 292 (34.5%)

# 查看可疑交互（原始 LLM 可能答错了）
dmr evolve suspects
# Output:
# [2024-03-15] daily:work "项目截止日期是什么时候" → 原始: "3月20日" → 投票: "3月25日" (3/3)
# [2024-03-16] daily:life "附近药店营业时间" → 原始: "9:00-21:00" → 投票: "8:30-22:00" (2/3)
```

### 配置

```toml
# config.toml

[evolve.labeling]
enabled = true
model = "qwen2.5:7b"       # 用于投票的模型
votes = 3                   # 每条交互跑几次
interval = "30s"            # 每条标注之间的间隔（避免占满 CPU）
schedule = "idle"           # "idle" = 只在系统空闲时跑 | "always" | "manual"
max_daily = 100             # 每天最多标注多少条（控制资源消耗）
```

## 效率分析

### 成本

| 操作 | 资源消耗 | 说明 |
|------|----------|------|
| 单条标注（3 次推理） | ~3-6s（7B） | 3 × 1-2s 推理 |
| 每日 100 条 | ~5-10 分钟 | 后台运行，不影响前台 |
| 存储 | ~1KB/条标签 | SQLite，可忽略 |
| 1 个月积累 | ~3000 条标签 | 足以建立可靠的 task_type → confidence 映射 |

### 收益

与 #25a 批判分析中指出的缺陷对照：

| 原始缺陷 | 投票标注如何缓解 |
|----------|----------------|
| Tier 0 缓存可能缓存错误响应 | 只有 high_confidence 才进缓存 |
| Tier 1 语义路由不可靠（0.70 阈值） | 替代为实证验证：有标注记录才路由 |
| 微调数据质量无法保证 | 只用 high/medium 标注的数据训练 |
| 用户沉默失败检测不到 | suspect 标签发现历史错误 |
| 不知道哪些任务适合本地模型 | ambiguous 比例量化任务难度 |

## 局限性

1. **需要 ollama 运行**：标注依赖本地模型，没有 ollama 就无法标注。但这与 Tier 1 共享基础设施，不额外增加依赖
2. **投票模型自身也有偏差**：如果 7B 模型在某类任务上系统性出错，投票会给出错误的 high_confidence。缓解：定期用 Tier 2（远程 LLM）抽检 high_confidence 标签的准确性
3. **开放性任务无法投票**：写邮件、写代码等创造性任务没有唯一正确答案，投票共识度天然低。这类任务会被标为 ambiguous——这其实是正确的分类，它们确实不适合 Tier 0/1
4. **推理成本**：每条交互 3 次推理，但离线运行、利用空闲算力，实际成本可接受

## 与其他 Issue 的关系

- **#25 自主进化架构**：投票标注是进化系统的**数据质量层**，解决 #25a 批判中指出的路由不可靠和数据质量问题
- **#24 Tape 标注系统**：共享 SQLite 存储和 CLI 基础设施，投票标注是自动化标注的一种
- **#18 Tape 数据挖掘**：标注数据为挖掘提供质量维度（只分析 high_confidence 的交互）
- **#23 动作重复检测**：投票机制与集成投票（ensemble）概念一致

## 实施建议

| 阶段 | 内容 | 依赖 |
|------|------|------|
| Phase 1 | SQLite schema + 离线标注 goroutine + CLI | #25 Phase 1（evolve 插件骨架） |
| Phase 2 | 标注结果 → Tier 0/1 路由决策集成 | #25 Phase 3（Tier 1 路由） |
| Phase 3 | Tier 2 抽检 + 标签准确性自校正 | Tier 2 可用 |

## 参考

- TapeAgents GAIA majority_vote：`examples/gaia_agent/README.md`（maj@3 数据）
- TapeAgents GAIA eval：`examples/gaia_agent/eval.py:150-156`（`action_repetitions()` + 投票）
- DMR #25 自主进化架构：`./25-self-evolution-architecture.md`
- DMR #25a 批判分析：`./25a-self-evolution-critique.md`
- DMR #24 Tape 标注：`./24-tape-browser-annotation.md`
