# 工具系统深入分析

## 概述

工具系统是 DMR 的扩展机制，允许 Agent 执行外部操作。

## 核心类型

### ToolSpec

```go
type ToolSpec struct {
    Name        string         // 工具名称 (lowerCamelCase)
    Description string         // 工具描述
    Parameters  map[string]any // JSON Schema 参数定义
    
    // 工具组
    Group ToolGroup  // "core", "extended", "mcp"
    
    // 加载控制
    AlwaysLoad bool   // 强制始终加载
    SearchHint string // 搜索关键词
}
```

### Tool

```go
type Tool struct {
    Spec               ToolSpec
    Handler            func(ctx *ToolContext, args map[string]any) (any, error)
    NeedContext        bool
    DynamicDescription DynamicDescriptionFunc
}
```

## 工具组

```go
const (
    ToolGroupCore     ToolGroup = "core"     // 核心工具，始终加载
    ToolGroupExtended ToolGroup = "extended" // 扩展工具，按需加载
    ToolGroupMCP      ToolGroup = "mcp"      // MCP 工具，默认延迟加载
)
```

### 工具加载行为

| Group | AlwaysLoad | 行为 |
|-------|------------|------|
| core | false | 始终加载 |
| core | true | 始终加载 |
| extended | false | 需要 discovery |
| extended | true | 始终加载 |
| mcp | false | 需要 discovery |
| mcp | true | 始终加载 |

## ToolContext

```go
type ToolContext struct {
    Ctx   context.Context
    Tape  string
    RunID string
    Meta  map[string]any
    State map[string]any
    Context map[string]any  // 插件传递的上下文
    
    // CWD 管理
    CwdManager *cwd.Manager
    CwdPolicy  CwdPolicy  // Allow, Track, Prevent
    CwdMustBeUnder string // 目录逃逸限制
}
```

## ToolExecutor

```go
type ToolExecutor struct {
    BeforeToolCall      BeforeToolCallFunc
    BatchBeforeToolCall BatchBeforeToolCallFunc
    Verbose             int
}
```

### 执行流程

```go
func (e *ToolExecutor) Execute(calls []core.ToolCallData, toolSet *ToolSet, ctx *ToolContext) *core.ToolExecution {
    // Phase 1: 解析所有调用
    resolved := make([]resolvedCall, len(calls))
    for i, call := range calls {
        t, ok := toolSet.Runnable[call.Function.Name]
        args, _ := parseArgs(call.Function.Arguments)
        resolved[i] = resolvedCall{tool: t, args: args}
    }
    
    // Phase 2: 批量策略检查
    denied := e.batchPolicyCheck(resolved, ctx)
    
    // Phase 3: 执行
    // 识别可并行的 subagent 调用
    subagentIndices := findSubagentCalls(calls, resolved, denied)
    
    if len(subagentIndices) > 1 {
        // 并行执行 subagent
        return e.executeWithParallelSubagents(...)
    } else {
        // 串行执行
        return e.executeSerial(...)
    }
}
```

### 并行 Subagent 执行

```go
func (e *ToolExecutor) executeWithParallelSubagents(calls, resolved, denied, subagentIndices, ctx, result) {
    // Fan-out: 并行启动所有 subagent
    var wg sync.WaitGroup
    resultCh := make(chan parallelResult, len(subagentIndices))
    
    for _, idx := range subagentIndices {
        wg.Add(1)
        go func(i int, rc resolvedCall) {
            defer wg.Done()
            out, err := rc.tool.Handler(ctx, rc.args)
            resultCh <- parallelResult{index: i, out: out, err: err}
        }(idx, resolved[idx])
    }
    
    // Gather: 等待所有完成
    go func() {
        wg.Wait()
        close(resultCh)
    }()
    
    subagentResults := make(map[int]parallelResult)
    for r := range resultCh {
        subagentResults[r.index] = r
    }
    
    // 按原始顺序填充结果
    for i, call := range calls {
        if _, ok := subagentSet[i]; ok {
            // 使用并行结果
            result.ToolResults = append(result.ToolResults, subagentResults[i].out)
        } else {
            // 串行执行
            out, _ := resolved[i].tool.Handler(ctx, resolved[i].args)
            result.ToolResults = append(result.ToolResults, out)
        }
    }
}
```

## 工具定义示例

### 简单工具

```go
func (p *MyPlugin) myTool() *tool.Tool {
    return &tool.Tool{
        Spec: tool.ToolSpec{
            Name:        "myTool",
            Description: "Does something useful",
            Group:       tool.ToolGroupCore,
            Parameters: map[string]any{
                "type": "object",
                "properties": map[string]any{
                    "arg1": map[string]any{
                        "type":        "string",
                        "description": "First argument",
                    },
                    "arg2": map[string]any{
                        "type":        "integer",
                        "default":     10,
                        "description": "Second argument",
                    },
                },
                "required": []string{"arg1"},
            },
        },
        Handler:     p.myHandler,
        NeedContext: true,
    }
}

func (p *MyPlugin) myHandler(ctx *tool.ToolContext, args map[string]any) (any, error) {
    arg1, _ := args["arg1"].(string)
    arg2 := 10
    if v, ok := args["arg2"].(float64); ok {
        arg2 = int(v)
    }
    
    // 使用 ctx
    workspace := ctx.State["_runtime_workspace"].(string)
    
    return fmt.Sprintf("Result: %s, %d", arg1, arg2), nil
}
```

### 动态描述

```go
func (p *ShellPlugin) shellTool() *tool.Tool {
    return &tool.Tool{
        Spec: tool.ToolSpec{
            Name:        "shell",
            Description: "Execute shell commands",
            // ...
        },
        DynamicDescription: func(ctx *tool.ToolContext) (string, error) {
            cwd := ctx.GetCwd()
            shell := os.Getenv("SHELL")
            return fmt.Sprintf("Execute shell commands. Current directory: %s, Shell: %s", cwd, shell), nil
        },
        Handler: p.execHandler,
    }
}
```

## 工具搜索

```go
func SearchTools(tools []*tool.Tool, query string) []*tool.Tool {
    words := splitWords(query)
    
    var matches []*tool.Tool
    for _, t := range tools {
        // 搜索名称、描述和 SearchHint
        searchable := strings.ToLower(t.Spec.Name + " " + t.Spec.Description + " " + t.Spec.SearchHint)
        if matchAnyWord(searchable, words) {
            matches = append(matches, t)
        }
    }
    return matches
}
```

## ToolSet

```go
type ToolSet struct {
    Schemas  []map[string]any  // OpenAI 格式工具模式
    Runnable map[string]*Tool  // 可执行工具映射
}

func NormalizeTools(tools []*tool.Tool) (*ToolSet, error) {
    ts := &ToolSet{
        Schemas:  make([]map[string]any, 0, len(tools)),
        Runnable: make(map[string]*Tool, len(tools)),
    }
    
    for _, t := range tools {
        if _, exists := ts.Runnable[t.Spec.Name]; exists {
            return nil, fmt.Errorf("duplicate tool name: %q", t.Spec.Name)
        }
        ts.Schemas = append(ts.Schemas, t.ToSchemaStatic())
        ts.Runnable[t.Spec.Name] = t
    }
    
    return ts, nil
}
```

## 工具命名规范

根据 `docs/tool-naming.md`:

- **格式**: `{pluginPrefix}{Action}`
- **风格**: lowerCamelCase
- **无点号**: 不使用 `.` 分隔

### 示例

| 工具名 | 插件 | 动作 |
|--------|------|------|
| `fsRead` | fs | Read |
| `fsWrite` | fs | Write |
| `tapeSearch` | tape | Search |
| `shellOutput` | shell | Output |
| `webFetch` | webtool | Fetch |
| `clawhubSearch` | clawhub | Search |

## 参数模式

### 基本类型

```go
map[string]any{
    "type": "object",
    "properties": map[string]any{
        "name":  map[string]any{"type": "string"},
        "count": map[string]any{"type": "integer"},
        "ratio": map[string]any{"type": "number"},
        "enabled": map[string]any{"type": "boolean"},
        "data": map[string]any{"type": "object"},
        "items": map[string]any{"type": "array", "items": map[string]any{"type": "string"}},
    },
    "required": []string{"name"},
}
```

### 复杂示例

```go
map[string]any{
    "type": "object",
    "properties": map[string]any{
        "cmd": map[string]any{
            "type":        "string",
            "description": "Command to execute",
        },
        "timeout_seconds": map[string]any{
            "type":        "integer",
            "default":     30,
            "description": "Timeout in seconds",
        },
        "background": map[string]any{
            "type":        "boolean",
            "default":     false,
            "description": "Run in background",
        },
        "env": map[string]any{
            "type": "object",
            "additionalProperties": map[string]any{"type": "string"},
            "description": "Environment variables",
        },
    },
    "required": []string{"cmd"},
}
```
