# P2: thinkFilter O(n^2) + 无界缓冲区

## 问题

`pkg/client/chat.go:650-686` 的 `thinkFilter` 有两个性能/安全问题：

### 1. O(n^2) 字符串匹配

```go
if f.inside {
    f.buf.WriteByte(ch)
    if strings.HasSuffix(f.buf.String(), "</think>") {  // ← 每个字节都转成 string 检查
        f.inside = false
        f.buf.Reset()
    }
    continue
}
```

`f.buf.String()` 在每个字节上都创建一个新 string。如果 think 块有 N 字节，这段代码的复杂度是 O(N^2)。对于 100K token 的 reasoning 输出，这会严重影响性能。

### 2. 无界缓冲区

当 `f.inside = true` 时，每个字节都写入 `f.buf`。如果 LLM 输出没有闭合的 `</think>` 标签（模型 bug 或流中断），缓冲区无限增长，最终 OOM。

## 修复方案

### 1. 用滑动窗口替代全缓冲区

```go
type thinkFilter struct {
    inside    bool
    buf       bytes.Buffer
    tailBuf   [8]byte  // "</think>" 长度 = 8
    tailLen   int
}

func (f *thinkFilter) feed(delta string) string {
    if !f.inside {
        // ... 正常的标签检测逻辑
    }
    
    // inside: 只维护尾部 8 字节的滑动窗口
    for i := 0; i < len(delta); i++ {
        if f.tailLen < 8 {
            f.tailBuf[f.tailLen] = delta[i]
            f.tailLen++
        } else {
            copy(f.tailBuf[:7], f.tailBuf[1:8])
            f.tailBuf[7] = delta[i]
        }
        
        if f.tailLen == 8 && string(f.tailBuf[:]) == "</think>" {
            f.inside = false
            f.tailLen = 0
            break
        }
    }
    return ""
}
```

### 2. 缓冲区上限

```go
const maxThinkBufferSize = 1 << 20  // 1MB

if f.inside && f.discardCount > maxThinkBufferSize {
    // 超过上限，强制退出 think 模式（可能是未闭合标签）
    log.Printf("[WARN] think block exceeded %d bytes without closing tag, force exit", maxThinkBufferSize)
    f.inside = false
}
```

## 参考

- DMR thinkFilter：`pkg/client/chat.go:642-695`
