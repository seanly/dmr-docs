# P2: 关键路径测试盲区 — 安全与可靠性核心组件缺乏测试

> 全代码库 55 个测试文件，覆盖了大部分核心包，但安全与可靠性关键路径存在测试空白：agent 主循环（错误处理分支）、webserver 插件（最有状态的插件零测试）、toolsearch（核心发现机制零测试）、外部插件 RPC 协议、cron 队列和存储。这些未测试组件恰好是本系列 issues 中多个设计缺陷的集中区域。

## 问题

### 已有测试的包（20+ 个包，55 个测试文件）

| 包 | 测试文件数 | 覆盖质量 |
|----|-----------|---------|
| `pkg/agent` | 9 | 好（compact、token 估算、截断） |
| `pkg/core` | 3 | 好（错误分类、执行引擎、结果类型） |
| `pkg/tape` | 6 | 好（存储、管理、查询、FTS5、时区） |
| `pkg/credentials` | 4 | 好（加密、存储、类型、配置集成） |
| `pkg/tool` | 3 | 中（context、executor、tool 定义） |
| `pkg/plugin` | 5 | 中（管理器、凭据、系统提示、外部调用） |
| `plugins/opapolicy` | 4 | 好（策略引擎、缓存、热重载、allow_rules） |
| 其他 | 21 | 分散在 config、auth、client、fs、shell 等 |

### 无测试的关键组件

| 组件 | 文件 | 为何关键 | 相关 Issue |
|------|------|---------|-----------|
| **Agent 主循环 `run()`** | `pkg/agent/loop.go` | 所有错误处理分支（tape 写入失败、LLM 连续失败、max steps、连续拒绝）未被直接测试 | #09, #12 |
| **webserver 插件** | `plugins/webserver/` | HTTP 服务器 + SSE + 认证 + mutex 状态管理 + 审批工作流，零测试 | — |
| **toolsearch 插件** | `plugins/toolsearch/` | 核心工具发现机制，rate limit 逻辑、DiscoveredToolsCleared reset 逻辑，零测试 | #45 |
| **外部插件 RPC** | `pkg/plugin/proto/` | 序列化/反序列化、连接中断、MuxBroker、反向 RPC 初始化，零测试 | #51 |
| **mcp 插件** | `plugins/mcp/` | MCP 协议桥接，3 个源文件，零测试 | — |
| **subagent 插件** | `plugins/subagent/` | 子 agent 编排，并行 fan-out，零测试 | — |
| **cron 队列 + 存储** | `plugins/cron/queue.go`, `storage_file.go` | 虚假持久化 (#50)、文件存储跨进程安全，零测试 | #50 |
| **credentials 插件** | `plugins/credentials/` | 凭据注入到工具参数的 hook 逻辑，零测试 | #49 |
| **service 包** | `pkg/service/` | launchd/systemd 服务管理，6 个源文件，零测试 | — |
| **powershell 插件** | `plugins/powershell/` | Windows 平台工具提供者，零测试 | — |
| **clawhub 插件** | `plugins/clawhub/` | 技能市场客户端，4 个源文件，零测试 | — |

### 交叉分析：未测试区域与已知缺陷重合

本系列 issues 中发现的设计缺陷**集中在无测试区域**：

```
#45 State 服务定位器   → toolsearch 中的 toolDiscoveryAgent 接口断言 → 零测试
#46 Hook 类型安全      → opapolicy 中的断言静默失败           → opapolicy 有测试但不覆盖此路径
#48 Plugin init 分级   → InitAll warn-and-continue          → manager 有测试但不覆盖 critical 路径
#49 Args 命名空间污染   → credentials 插件 hook 逻辑          → 零测试
#50 Cron 死锁          → applyJobs + queueImmediateRun 链     → 零测试
#51 外部插件重连       → RPC 连接断开后的行为                 → 零测试
```

### Agent Loop 的未测试错误分支

`pkg/agent/loop.go` 的 `run()` 方法包含以下未测试的分支：

1. **Tape append 失败继续运行**（多处 `slog.Warn`）
2. **预防性 compact 失败后仍调用 API**（line 183）
3. **反应式 context overflow 处理**（line 210-221）
4. **连续 2 轮全拒绝后停止**（line 346-357）
5. **Proactive auto-handoff + anchor fallback**（line 366-401）
6. **Max steps 耗尽**（line 415）

虽然 `pkg/agent/` 有 9 个测试文件，但它们测试的是独立的辅助功能（compact prompt 生成、token 估算、结果截断），不是 `run()` 主循环的集成行为。

## 设计方案

### 优先级排序

按照"缺陷严重度 x 测试缺失程度"排序：

**P0（必须测试）**:
1. Agent loop `run()` 的错误处理分支 — 关键路径，所有 bug 汇聚于此
2. External plugin RPC 错误场景 — 安全关键（凭据传输）

**P1（应该测试）**:
3. Cron 死锁触发路径 — 确定性 bug，一行测试即可验证
4. toolsearch 的 rate limit + reset 逻辑
5. credentials 插件 hook 的注入和清理

**P2（期望测试）**:
6. webserver 基本路由和 SSE
7. mcp 桥接
8. subagent fan-out 并行

### Agent Loop 测试策略

Agent loop 难以测试是因为它依赖 LLM API。建议使用 mock + 编排模式：

```go
// pkg/agent/loop_test.go

func TestRun_TapeAppendFailure(t *testing.T) {
    // 构造：tape store 的 Append 返回错误
    store := &failingTapeStore{appendErr: fmt.Errorf("disk full")}
    agent := newTestAgent(t, withTapeStore(store), withMockLLM(respondText("hello")))

    // 执行
    result, err := agent.Run(ctx, "test", "hi")

    // 断言：run 应该成功完成（当前行为：warn-and-continue）
    // 一旦 #09 修复为 fail-fast，此测试应改为 assert error
    assert.NoError(t, err)
    assert.Equal(t, "hello", result)
}

func TestRun_ConsecutiveDenials(t *testing.T) {
    // 构造：OPA 拒绝所有工具调用
    agent := newTestAgent(t,
        withMockLLM(respondToolCalls("shell", map[string]any{"command": "ls"})),
        withPolicy(denyAll()),
    )

    result, err := agent.Run(ctx, "test", "list files")

    // 断言：2 轮全拒绝后应停止
    assert.ErrorContains(t, err, "consecutive denials")
}
```

### Cron 死锁测试

```go
// plugins/cron/cron_test.go

func TestApplyJobs_ImmediateRun_NoDeadlock(t *testing.T) {
    p := newTestPlugin(t)
    p.Init(ctx, map[string]any{})

    // 添加一个 run_on_load 的 job
    job := &Job{Name: "test", Schedule: "@immediately", RunOnLoad: true}

    done := make(chan struct{})
    go func() {
        p.applyJobs()  // 如果死锁，此 goroutine 会永远阻塞
        close(done)
    }()

    select {
    case <-done:
        // 成功
    case <-time.After(5 * time.Second):
        t.Fatal("deadlock detected in applyJobs")
    }
}
```

## 实现步骤

- [ ] 创建 `pkg/agent/loop_test.go`，覆盖 6 个错误处理分支
- [ ] 创建 Agent loop 测试辅助工具（mock LLM、mock tape store、mock policy）
- [ ] 创建 `plugins/cron/deadlock_test.go`，验证 immediate job 路径
- [ ] 创建 `plugins/toolsearch/toolsearch_test.go`，覆盖 rate limit 和 reset
- [ ] 创建 `plugins/credentials/hook_test.go`，覆盖 credential_bindings 注入
- [ ] 创建 `pkg/plugin/proto/rpc_test.go`，覆盖连接断开和反序列化错误
- [ ] 为 `plugins/webserver/` 添加基本路由测试

## 参考

- `pkg/agent/loop.go` — Agent 主循环（6 个未测试分支）
- `plugins/toolsearch/toolsearch.go` — 零测试
- `plugins/webserver/` — 零测试
- `plugins/cron/queue.go` — 虚假持久化，零测试
- `pkg/plugin/proto/rpc.go` — RPC 协议，零测试
- 本系列所有相关 issues：#09, #45, #46, #48, #49, #50, #51
