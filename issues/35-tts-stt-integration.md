# P3: TTS/STT 语音交互 — 外部插件或 MCP 集成

> 灵感来源：Hermes Agent `text_to_speech`（Edge TTS 免费 + ElevenLabs 高级）+ `faster-whisper` STT
>
> 语音交互是面向无障碍场景和免手操作场景的远期能力，优先级较低但架构上需要预留。

## 问题

DMR 当前只有文本交互界面（CLI + Web UI），不支持语音输入/输出：

| 场景 | 当前行为 | 期望行为 |
|------|---------|---------|
| 驾车/运动中使用 Agent | 无法操作 | 语音输入指令，语音播报结果 |
| 视力障碍用户 | 只能依赖屏幕阅读器 | 原生语音支持 |
| 长文本输出 | 只能阅读 | 可选语音播报关键结论 |
| 多语言场景 | 键盘输入不便 | 语音输入更自然 |

## Hermes Agent 的做法

### TTS（文本转语音）

```python
# hermes_agent 依赖
# edge-tts>=7.2.7    — 免费（微软 Edge 在线 TTS）
# elevenlabs>=1.0    — 付费（高质量语音合成）

# 工具：text_to_speech
# - 接受文本 + 语音选择
# - 生成 MP3 音频文件
# - 在 CLI 中直接播放或保存
```

### STT（语音转文本）

```python
# hermes_agent 依赖（可选）
# faster-whisper>=1.0.0 — 本地 Whisper 推理

# 用于：
# - 语音消息处理（Telegram 语音、Discord 语音）
# - 本地麦克风输入（CLI 模式）
```

## DMR 的实现方案

### 方案对比

| 方案 | 开发成本 | 灵活性 | 独立性 |
|------|---------|--------|--------|
| **A: MCP 桥接** | 零代码 | 高（依赖 MCP server 生态） | 需要外部 MCP server |
| **B: 外部插件 RPC** | 中等 | 最高（完全自定义） | 需要独立进程 |
| **C: 内建插件** | 高（Go 生态 TTS/STT 库少） | 低 | 自包含 |

**推荐：方案 A（MCP）作为首选，方案 B 作为高级备选。**

### 方案 A：MCP 桥接

#### TTS via MCP

```toml
# ~/.dmr/config.toml

[plugins.config.servers.tts]
command = "python3"
args = ["-m", "mcp_tts_server"]
```

需要一个轻量的 MCP TTS server（Python 实现）：

```python
# mcp_tts_server/__main__.py

import asyncio
import edge_tts
from mcp import Server, Tool

app = Server("tts")

@app.tool("textToSpeech")
async def text_to_speech(text: str, voice: str = "zh-CN-XiaoxiaoNeural",
                          output: str = "/tmp/tts_output.mp3") -> str:
    """Convert text to speech audio file using Edge TTS (free)."""
    communicate = edge_tts.Communicate(text, voice)
    await communicate.save(output)
    return f"Audio saved to {output}"

@app.tool("listVoices")
async def list_voices(language: str = "zh") -> str:
    """List available TTS voices for a language."""
    voices = await edge_tts.list_voices()
    filtered = [v for v in voices if v["Locale"].startswith(language)]
    return "\n".join(f"{v['ShortName']}: {v['FriendlyName']}" for v in filtered[:20])

if __name__ == "__main__":
    app.run()
```

安装：

```bash
pip install edge-tts mcp
```

#### STT via MCP

```toml
[plugins.config.servers.stt]
command = "python3"
args = ["-m", "mcp_stt_server"]
```

```python
# mcp_stt_server/__main__.py

from faster_whisper import WhisperModel
from mcp import Server, Tool

app = Server("stt")
model = WhisperModel("base", device="cpu")  # 或 "cuda"

@app.tool("speechToText")
async def speech_to_text(audio_path: str, language: str = "zh") -> str:
    """Transcribe audio file to text using Whisper."""
    segments, info = model.transcribe(audio_path, language=language)
    text = " ".join(segment.text for segment in segments)
    return text

if __name__ == "__main__":
    app.run()
```

安装：

```bash
pip install faster-whisper mcp
```

### 方案 B：外部插件 RPC（高级）

如果需要更深度的集成（如实时流式 TTS、麦克风输入），可以实现为 DMR 外部插件：

```go
// 外部插件接口（Go 或 Python，通过 go-plugin RPC）

// 工具暴露:
// - ttsSpeak(text, voice, output_path) → 生成音频文件
// - ttsStream(text, voice) → 流式音频（实时播放）
// - sttTranscribe(audio_path, language) → 文本
// - sttListen(duration_sec) → 实时录音 + 转写
```

外部插件配置：

```toml
[[plugins]]
name = "voice"
enabled = true
path = "~/.dmr/plugins/voice-plugin"  # 独立可执行文件

[plugins.config]
tts_engine = "edge-tts"           # edge-tts | elevenlabs
tts_voice = "zh-CN-XiaoxiaoNeural"
stt_model = "base"                # tiny | base | small | medium | large
stt_device = "cpu"                # cpu | cuda
```

### OPA 策略

TTS/STT 涉及音频文件读写，建议配置策略：

```rego
# 允许 TTS 输出到 /tmp
allow if {
    input.tool == "mcp_tts_textToSpeech"
    startswith(input.args.output, "/tmp/")
}

# STT 只允许读取特定目录
allow if {
    input.tool == "mcp_stt_speechToText"
    startswith(input.args.audio_path, "/tmp/")
}
```

## 解决的问题

- **无障碍访问**：视力障碍用户可以通过语音与 Agent 交互
- **免手操作**：驾车、运动、做饭等场景下的语音控制
- **多语言输入**：语音输入比键盘更自然（特别是 CJK 语言）
- **长文本播报**：Agent 的长回复可以语音播报关键结论

## 代价与风险

| 代价 | 评估 |
|------|------|
| **外部依赖** | Python + edge-tts + faster-whisper |
| **资源消耗** | Whisper base 模型占用 ~300MB 内存；edge-tts 几乎无消耗 |
| **延迟** | TTS: \<2 秒；STT (base): \<5 秒（本地 CPU） |
| **成本** | Edge TTS 免费；ElevenLabs 按量计费 |
| **维护** | MCP server 需要独立维护（但逻辑极简） |

## 实施路线

### Phase 1：TTS（文本转语音）
- MCP TTS server（edge-tts）
- 配置示例和文档
- 基本的中英文语音支持

### Phase 2：STT（语音转文本）
- MCP STT server（faster-whisper）
- 音频文件转写
- 与 CLI 集成（语音输入模式）

### Phase 3：实时语音交互
- 外部插件 RPC（流式 TTS + 实时录音）
- Web UI 集成（WebSocket 音频流）
- 语音唤醒（可选）

## 参考

- Hermes Agent edge-tts 集成：`pyproject.toml` 依赖声明
- Hermes Agent faster-whisper：`pyproject.toml` 可选依赖
- DMR MCP 插件：`plugins/mcp/mcp.go`
- DMR MCP 桥接：`plugins/mcp/bridge.go`
- DMR 外部插件机制：`pkg/plugin/external.go:17-136`
- DMR 外部插件加载：`pkg/plugin/loader.go:27-94`
- Edge TTS：`https://github.com/rany2/edge-tts`
- Faster Whisper：`https://github.com/SYSTRAN/faster-whisper`
