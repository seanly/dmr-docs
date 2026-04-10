# P2: LLM 响应缓存（LLM Response Cache）

> 灵感来源：TapeAgents `CachedLLM`（MD5 hash key + JSONL 文件存储）

## 问题

DMR 没有 LLM 响应缓存。开发迭代中：

- 调试 system prompt 时，反复测试相同的问题，每次都真实调用 API
- 开发新插件/工具时，同样的工具调用序列重复执行
- 演示时需要稳定的响应，但 LLM 输出不可复现
- 离线开发（如飞机/火车上）无法工作

## TapeAgents 的做法

```python
class CachedLLM(LLM):
    def get_prompt_key(self, prompt):
        # 排除不稳定字段，序列化后 MD5
        data = prompt.model_dump(exclude={"id"})
        return hashlib.md5(json.dumps(data, sort_keys=True).encode()).hexdigest()
    
    def generate(self, prompt):
        key = self.get_prompt_key(prompt)
        if key in self.cache:
            yield from self.cache[key]  # 缓存命中
        else:
            events = list(self._generate(prompt))  # 真实调用
            self.cache[key] = events
            self._save_to_file(key, events)
            yield from events
```

缓存文件：JSONL 格式，`llm_cache_{model}_{param_hash}.{pid}.{tid}.jsonl`，多进程安全。

## DMR 的实现方案

### 装饰器模式 — 不改核心代码

```go
// pkg/core/cache.go

type CachedProvider struct {
    inner ChatProvider
    store CacheStore
    model string
}

type CacheStore interface {
    Get(key string) (*ChatResponse, bool)
    Put(key string, resp *ChatResponse)
    Close() error
}

func (c *CachedProvider) CreateChatCompletion(
    ctx context.Context, req ChatRequest,
) (ChatResponse, error) {
    key := cacheKey(c.model, req)
    
    if resp, ok := c.store.Get(key); ok {
        return *resp, nil
    }
    
    resp, err := c.inner.CreateChatCompletion(ctx, req)
    if err == nil {
        c.store.Put(key, &resp)
    }
    return resp, err
}

func cacheKey(model string, req ChatRequest) string {
    // 只对稳定字段做 hash
    stable := struct {
        Model    string         `json:"model"`
        Messages []any          `json:"messages"`
        Tools    []any          `json:"tools,omitempty"`
    }{
        Model:    model,
        Messages: req.Messages,
        Tools:    req.Tools,
    }
    data, _ := json.Marshal(stable)
    h := sha256.Sum256(data)
    return hex.EncodeToString(h[:16])  // 前 128 bit 足够
}
```

### 存储后端

简单起见用 SQLite（DMR 已有依赖）：

```go
// pkg/core/cache_sqlite.go

type SQLiteCacheStore struct {
    db *sql.DB
}

// Schema
// CREATE TABLE llm_cache (
//     key        TEXT PRIMARY KEY,
//     model      TEXT,
//     response   TEXT,     -- JSON
//     created_at TEXT,
//     hit_count  INTEGER DEFAULT 0
// );

func (s *SQLiteCacheStore) Get(key string) (*ChatResponse, bool) {
    var respJSON string
    err := s.db.QueryRow(
        "UPDATE llm_cache SET hit_count = hit_count + 1 WHERE key = ? RETURNING response",
        key,
    ).Scan(&respJSON)
    if err != nil { return nil, false }
    
    var resp ChatResponse
    json.Unmarshal([]byte(respJSON), &resp)
    return &resp, true
}
```

### 启用方式

```toml
# dmr.toml
[agent]
llm_cache = false  # 默认关闭

[agent.cache]
db_path = "~/.dmr/llm_cache.db"
ttl = "7d"         # 缓存过期时间
max_size = "500MB" # 最大缓存大小
```

```bash
# 命令行开关（覆盖配置）
dmr chat --cache           # 启用缓存
dmr run --cache "prompt"   # 单次启用

# 缓存管理
dmr cache stats            # 命中率、大小
dmr cache clear            # 清空
dmr cache clear --model gpt-4o  # 清空特定模型
```

### 流式响应处理

流式缓存需要特殊处理 — 缓存完整响应，重放时模拟流式：

```go
func (c *CachedProvider) CreateChatCompletionStream(
    ctx context.Context, req ChatRequest,
) (<-chan StreamChunk, error) {
    key := cacheKey(c.model, req)
    
    if resp, ok := c.store.Get(key); ok {
        // 将完整响应拆成 chunk 模拟流式
        ch := make(chan StreamChunk, 1)
        go func() {
            defer close(ch)
            ch <- StreamChunk{Text: resp.Text, Done: true}
        }()
        return ch, nil
    }
    
    // 真实流式调用，收集完整响应后缓存
    innerCh, err := c.inner.CreateChatCompletionStream(ctx, req)
    if err != nil { return nil, err }
    
    outCh := make(chan StreamChunk)
    go func() {
        defer close(outCh)
        var fullResp ChatResponse
        for chunk := range innerCh {
            outCh <- chunk
            fullResp.Text += chunk.Text
            // 收集 tool_calls 等
        }
        c.store.Put(key, &fullResp)
    }()
    return outCh, nil
}
```

## 解决的问题

- **开发成本**：调试迭代中节省 API 费用
- **开发速度**：缓存命中时响应瞬间返回，无网络延迟
- **离线开发**：缓存过的交互可在离线环境重放
- **演示稳定性**：相同 prompt 得到相同响应

## 代价与风险

- 缓存失效：prompt 模板变更后旧缓存可能产生不正确的结果
  - 缓解：cache key 包含 model + messages + tools，模板变更自动 miss
  - 缓解：`dmr cache clear` 手动清理
- 不适合生产：生产环境中相同 prompt 可能需要不同响应（时效性数据等）
  - 缓解：默认关闭，需显式启用
- 流式缓存增加复杂度
  - 缓解：第一版可以只缓存非流式，流式 fallback 到真实调用

## 可扩展性

中。缓存层是装饰器模式，不侵入核心。未来可扩展：
- 缓存预热（从录制的测试数据加载）
- 与 P0 重放测试复用存储格式
- 部分匹配（messages 前缀相同时复用）

## 参考

- TapeAgents `CachedLLM`：`tapeagents/llms/cached.py`
- DMR `ChatProvider` 接口：`pkg/core/execution.go:20-23`
- DMR `LLMCore.SetClientForModel`：`pkg/core/execution.go:459`
