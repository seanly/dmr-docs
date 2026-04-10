# P1: Hook 系统无 panic 恢复 + 切片竞争

> 灵感来源：TapeAgents `execute_agent()` 的顶层异常捕获 + `FatalError` 分级机制

## 问题 1：Hook panic 导致进程崩溃

`pkg/plugin/hooks.go:87-103`：

```go
for _, e := range entries {
    result, err := e.fn(ctx, args...)  // ← 如果 panic，整个进程崩溃
    if err != nil {
        return results, err
    }
}
```

Hook 来自插件（包括外部进程插件），任何插件 bug 都可以 panic 掉整个 DMR。在 `CallAll` 策略下尤其危险 — 前几个 hook 的副作用已经提交，后续 hook 因 panic 未执行，系统处于不一致状态。

## 问题 2：Hook 切片并发读写竞争

`hooks.go:88-89`：

```go
r.mu.RLock()
entries := r.hooks[hookName]  // ← 获取切片引用
r.mu.RUnlock()
// 锁已释放，但 entries 和 r.hooks[hookName] 共享底层数组
```

如果另一个 goroutine 同时调用 `RegisterHook()`（执行 `append` + `sort.Slice`），`sort.Slice` 会**原地重排**底层数组元素，而此处正在遍历同一个数组。这是一个真实的数据竞争。

## TapeAgents 的做法

TapeAgents 没有 hook 系统，但 `execute_agent()` 提供了一个优秀的错误边界模式：

```python
try:
    for event in main_loop(agent, start_tape, environment, max_loops=max_loops):
        ...
except Exception as e:
    tape_error = f"Agent loop exception: {e}"
    logger.exception(...)
# 无论如何都返回带有 error 标注的 tape
```

TapeAgents 的 `ToolEnvironment` 也区分 `FatalError`（必须中止）和普通 `Exception`（降级为字符串内容）：

```python
try:
    content = tool.run(tool_input=args)
except FatalError as e:
    raise e
except Exception as e:
    content = str(e)  # 降级：错误变成观察内容
```

## 修复方案

### 1. Hook 执行加 panic 恢复

```go
func (r *HookRegistry) callHook(e hookEntry, ctx context.Context, args ...any) (result any, err error) {
    defer func() {
        if p := recover(); p != nil {
            err = fmt.Errorf("hook %q panicked: %v\n%s", e.plugin, p, debug.Stack())
            log.Printf("[ERROR] %v", err)
        }
    }()
    return e.fn(ctx, args...)
}

func (r *HookRegistry) CallAll(ctx context.Context, hookName string, args ...any) ([]any, error) {
    // ...
    for _, e := range entries {
        result, err := r.callHook(e, ctx, args...)
        if err != nil {
            // 区分 fatal 和 non-fatal
            if isFatal(err) {
                return results, err
            }
            log.Printf("[WARN] hook %s/%s failed: %v", e.plugin, hookName, err)
            continue  // 非致命错误：跳过这个 hook，继续执行后续 hook
        }
        results = append(results, result)
    }
    return results, nil
}
```

### 2. Hook 切片安全拷贝

```go
func (r *HookRegistry) getEntries(hookName string) []hookEntry {
    r.mu.RLock()
    defer r.mu.RUnlock()
    
    src := r.hooks[hookName]
    // 返回独立副本，不共享底层数组
    dst := make([]hookEntry, len(src))
    copy(dst, src)
    return dst
}
```

### 3. RegisterHook 时也拷贝

```go
func (r *HookRegistry) RegisterHook(hookName string, ...) {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    // append 到新切片，不修改旧切片
    newEntries := make([]hookEntry, len(r.hooks[hookName])+1)
    copy(newEntries, r.hooks[hookName])
    newEntries[len(newEntries)-1] = hookEntry{...}
    sort.Slice(newEntries, ...)
    r.hooks[hookName] = newEntries  // 替换整个切片
}
```

## 参考

- TapeAgents `execute_agent` 异常边界：`tapeagents/orchestrator.py:308-329`
- TapeAgents `FatalError` 分级：`tapeagents/environment.py:148-149`
- DMR hook 执行：`pkg/plugin/hooks.go:87-103`
- DMR hook 注册：`pkg/plugin/hooks.go:58-68`
