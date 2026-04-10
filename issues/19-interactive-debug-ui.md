# P2: 交互式调试 UI — Tape Surgery

> 灵感来源：TapeAgents Studio（Gradio）的 tape surgery、流式重执行、agent 热重载

## TapeAgents 的做法

TapeAgents Studio 提供了一个 Gradio Web UI，核心能力：

### 1. Tape Surgery — 修改执行轨迹并重跑

```python
# tapeagents/studio.py
def pop_step(tape):
    """移除 tape 最后一步，从前一步重新执行"""
    return tape.model_copy(update={"steps": tape.steps[:-1]})

def keep_steps(tape, n):
    """只保留前 n 步，从第 n 步重新执行"""
    return tape.model_copy(update={"steps": tape.steps[:n]})
```

**使用场景**：agent 在第 5 步做了一个错误的工具调用。用户可以：
1. 在 UI 中 `keep_steps(tape, 4)` — 保留前 4 步
2. 手动修改第 4 步的内容（或让 LLM 从这里重新决策）
3. 点击"Resume" → agent 从第 4 步之后继续执行

### 2. 流式重执行 — 实时观察 agent 思考过程

```python
# Studio 中的流式执行
for event in main_loop(agent, start_tape, env):
    if event.agent_event:
        # 实时显示 agent 的每一步
        render_step(event.agent_event.step)
    if event.env_event:
        # 实时显示环境反馈（工具执行结果）
        render_observation(event.env_event.observation)
```

### 3. Agent 热重载 — 修改 agent 代码后立即重跑

Studio 支持修改 agent 的 node 定义、prompt 模板后，不需要重启服务就能重新执行。

### 4. Renderer 层级

```
BasicRenderer    → 纯文本 step dump
  ↓
PrettyRenderer   → 折叠/展开 step 详情、语法高亮
  ↓
CameraReadyRenderer → 带媒体嵌入（图片、视频）、导出为 HTML/PDF
```

## DMR 可以学什么

DMR 当前的调试手段：**看日志**。没有结构化的 tape 回放、没有 surgery、没有可视化。

### 方案：`dmr debug` CLI + 可选 Web UI

#### 阶段一：CLI 调试工具（低成本、高价值）

```go
// cmd/debug.go

// 1. Tape 回放 — 步骤级查看
// dmr debug replay <tape-name>
func replayTape(tapeName string) {
    entries, _ := store.FetchAll(tapeName, nil)
    for i, entry := range entries {
        fmt.Printf("─── Step %d [%s] ───\n", i, entry.Kind)
        switch entry.Kind {
        case "user":
            printColored(Green, entry.Payload["content"])
        case "tool_call":
            printColored(Yellow, formatToolCall(entry))
        case "tool_result":
            printColored(Cyan, truncate(entry.Payload["content"], 500))
        case "assistant":
            printColored(White, entry.Payload["content"])
        }
        
        // 交互式：按 Enter 继续，输入 's' 跳转，'q' 退出
        cmd := waitForInput()
        if cmd == "q" { break }
    }
}

// 2. Tape 对比 — diff 两次执行
// dmr debug diff <tape-a> <tape-b>
func diffTapes(tapeA, tapeB string) {
    entriesA, _ := store.FetchAll(tapeA, nil)
    entriesB, _ := store.FetchAll(tapeB, nil)
    
    // 对齐相同的 user message，对比 agent 的不同决策
    alignedPairs := alignByUserMessages(entriesA, entriesB)
    for _, pair := range alignedPairs {
        if pair.A.Kind == "tool_call" && pair.B.Kind == "tool_call" {
            if pair.A.Payload["name"] != pair.B.Payload["name"] {
                fmt.Printf("DIVERGE at step %d: %s vs %s\n",
                    pair.Step, pair.A.Payload["name"], pair.B.Payload["name"])
            }
        }
    }
}

// 3. Tape Surgery — 截断并重跑
// dmr debug surgery <tape-name> --keep-steps 4 --resume
func surgeryAndResume(tapeName string, keepSteps int) {
    entries, _ := store.FetchAll(tapeName, nil)
    truncated := entries[:keepSteps]
    
    // 创建新的 tape，从截断处恢复
    newTape := fmt.Sprintf("%s-debug-%d", tapeName, time.Now().Unix())
    for _, e := range truncated {
        store.Append(newTape, e)
    }
    
    // 用相同的 agent 配置，从截断处继续执行
    agent := loadAgentConfig()
    agent.Resume(newTape, keepSteps)
}
```

#### 阶段二：Web UI（可选，用 Go template 或嵌入式前端）

```go
// pkg/debug/server.go
func StartDebugServer(store TapeStore, port int) {
    http.HandleFunc("/tapes", handleListTapes)        // 列出所有 tape
    http.HandleFunc("/tape/{name}", handleViewTape)    // 查看单条 tape
    http.HandleFunc("/tape/{name}/step/{n}", handleStep) // 查看单步详情
    http.HandleFunc("/tape/{name}/surgery", handleSurgery) // surgery 操作
    http.HandleFunc("/tape/{name}/resume", handleResume)   // 从截断处重跑
    
    // 实时流式输出（SSE）
    http.HandleFunc("/tape/{name}/stream", handleStream)
    
    log.Printf("Debug UI at http://localhost:%d", port)
    http.ListenAndServe(fmt.Sprintf(":%d", port), nil)
}
```

#### 渲染层

```go
// pkg/debug/renderer.go

type Renderer interface {
    RenderStep(entry TapeEntry) string
}

type TextRenderer struct{}    // CLI 用：彩色文本
type JSONRenderer struct{}    // API 用：结构化 JSON
type HTMLRenderer struct{}    // Web UI 用：HTML 卡片
```

## 为什么值得做

| 维度 | 说明 |
|------|------|
| **解决什么问题** | Agent 出错时，用户只能翻日志猜测；无法快速定位决策分歧点 |
| **关键场景** | Prompt 调优、工具配置调试、失败分析、团队协作 review |
| **阶段一成本** | 低（CLI 工具，几百行 Go，无外部依赖） |
| **阶段二成本** | 中（嵌入式 Web UI，可用 Go template 或引入轻量前端） |
| **DMR 特有价值** | DMR 面向 DevOps 团队，debugging 是日常需求；生产环境的 tape 可以被 "带回" 本地 debug |
| **可扩展性** | CLI → Web UI → VS Code 插件 → 团队共享 debug 平台 |

## 参考

- TapeAgents Studio：`tapeagents/studio.py`
- TapeAgents Renderers：`tapeagents/renderers/`
- DMR Tape 存储：`pkg/tape/store.go`, `pkg/tape/sqlite_store.go`
- DMR CLI 入口：`cmd/`
