# P0: 视觉分析插件 — 图像理解与截图分析

> 灵感来源：Hermes Agent `vision_analyze` 工具 + Claude Vision API + GPT-4o 视觉能力
>
> 视觉分析是 Agent 从"纯文本工具"进化为"多模态助手"的第一步，也是 ROI 最高的多模态能力。

## 问题

DMR 当前**完全没有视觉能力**：

| 场景 | 当前行为 | 期望行为 |
|------|---------|---------|
| 用户说"看看这个截图的报错" | 无法处理 | 分析截图，提取错误信息 |
| 需要理解 UI 布局 | 只能看 HTML 源码 | 直接分析渲染后的视觉效果 |
| 分析图表/架构图 | 无法理解 | 提取关键信息和关系 |
| OCR 提取文本 | 不支持 | 从图片中提取文字 |

主流 LLM（Claude、GPT-4o、Gemini）都已原生支持视觉输入。DMR 需要的只是一个桥接层。

## Hermes Agent 的做法

```python
# hermes_agent/tools/vision_tools.py

# vision_analyze 工具：
# 1. 接受图片路径或 URL
# 2. base64 编码（本地文件）或直接传 URL
# 3. 调用支持视觉的 LLM（Claude Vision / GPT-4o）
# 4. 使用独立的辅助 LLM 客户端（可配置为更便宜的模型）
# 5. 返回分析文本

# 关键设计：vision 使用独立的 auxiliary_client
# 不占用主对话的上下文窗口
# 可配置为不同于主模型的视觉模型
```

Hermes 还有 `browser_vision`（截取浏览器页面后分析）和 `image_generate`（生成图片），但视觉分析是核心。

## DMR 的实现方案

### 整体架构

实现为内建插件，通过 OpenAI 兼容 API 调用视觉模型（大多数提供商都支持 OpenAI 格式的 image_url 消息）。

```
plugins/vision/
    vision.go        // 插件入口
    analyze.go       // 图像分析逻辑
    encode.go        // 图片编码（base64、URL 检测）
```

### 1. 插件骨架

```go
// plugins/vision/vision.go

package vision

type VisionPlugin struct {
    config VisionConfig
    client *http.Client
}

type VisionConfig struct {
    Model   string `mapstructure:"model"`    // 视觉模型，默认 "gpt-4o"
    APIKey  string `mapstructure:"api_key"`  // 可独立配置，或复用主模型
    APIBase string `mapstructure:"api_base"` // 默认 https://api.openai.com/v1
    MaxTokens int  `mapstructure:"max_tokens"` // 视觉分析输出上限，默认 1024
}

func (p *VisionPlugin) Name() string    { return "vision" }
func (p *VisionPlugin) Version() string { return "0.1.0" }

func (p *VisionPlugin) Init(ctx context.Context, config map[string]any) error {
    p.config = parseConfig(config)
    if p.config.Model == "" {
        p.config.Model = "gpt-4o"
    }
    if p.config.MaxTokens == 0 {
        p.config.MaxTokens = 1024
    }
    p.client = &http.Client{Timeout: 60 * time.Second}
    return nil
}

func (p *VisionPlugin) RegisterHooks(registry *plugin.HookRegistry) {
    registry.RegisterExtendedTools(p.Name(), 0, p.registerTools)
}
```

### 2. 工具定义

```go
// plugins/vision/vision.go (续)

func (p *VisionPlugin) registerTools(ctx context.Context, args ...any) (any, error) {
    return []*tool.Tool{
        {
            Name: "visionAnalyze",
            Description: `Analyze an image using a vision-capable LLM.
Accepts local file paths (PNG, JPG, GIF, WebP) or HTTP(S) URLs.
Use for: screenshot analysis, error message extraction, UI review,
chart interpretation, architecture diagram understanding, OCR.`,
            Parameters: map[string]any{
                "type": "object",
                "properties": map[string]any{
                    "image": map[string]any{
                        "type":        "string",
                        "description": "Local file path or HTTP(S) URL of the image",
                    },
                    "prompt": map[string]any{
                        "type":        "string",
                        "description": "What to analyze or extract from the image",
                    },
                },
                "required": []string{"image", "prompt"},
            },
            Handler:    p.analyzeHandler,
            Group:      tool.ToolGroupExtended,
            SearchHint: "image, screenshot, visual, OCR, picture, photo, 截图, 图片, 图像",
        },
    }, nil
}
```

### 3. 图像分析核心逻辑

```go
// plugins/vision/analyze.go

func (p *VisionPlugin) analyzeHandler(ctx *tool.ToolContext, args map[string]any) (any, error) {
    imagePath, _ := args["image"].(string)
    prompt, _ := args["prompt"].(string)

    if prompt == "" {
        prompt = "Describe this image in detail."
    }

    // 构建 image_url content part
    imageContent, err := p.resolveImage(imagePath)
    if err != nil {
        return map[string]any{"error": fmt.Sprintf("failed to load image: %v", err)}, nil
    }

    // 构建 OpenAI Vision API 请求
    messages := []map[string]any{
        {
            "role": "user",
            "content": []map[string]any{
                {
                    "type": "text",
                    "text": prompt,
                },
                imageContent,
            },
        },
    }

    reqBody := map[string]any{
        "model":      p.config.Model,
        "messages":   messages,
        "max_tokens": p.config.MaxTokens,
    }

    body, _ := json.Marshal(reqBody)
    apiURL := strings.TrimRight(p.config.APIBase, "/") + "/chat/completions"
    req, _ := http.NewRequestWithContext(ctx.Context, "POST", apiURL, bytes.NewReader(body))
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Authorization", "Bearer "+p.config.APIKey)

    resp, err := p.client.Do(req)
    if err != nil {
        return map[string]any{"error": fmt.Sprintf("API request failed: %v", err)}, nil
    }
    defer resp.Body.Close()

    var result struct {
        Choices []struct {
            Message struct {
                Content string `json:"content"`
            } `json:"message"`
        } `json:"choices"`
        Error *struct {
            Message string `json:"message"`
        } `json:"error"`
    }

    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return map[string]any{"error": fmt.Sprintf("failed to parse response: %v", err)}, nil
    }

    if result.Error != nil {
        return map[string]any{"error": result.Error.Message}, nil
    }

    if len(result.Choices) == 0 {
        return map[string]any{"error": "no response from vision model"}, nil
    }

    return map[string]any{
        "analysis": result.Choices[0].Message.Content,
        "model":    p.config.Model,
        "image":    imagePath,
    }, nil
}
```

### 4. 图像编码

```go
// plugins/vision/encode.go

import (
    "encoding/base64"
    "mime"
    "path/filepath"
)

var supportedFormats = map[string]string{
    ".png":  "image/png",
    ".jpg":  "image/jpeg",
    ".jpeg": "image/jpeg",
    ".gif":  "image/gif",
    ".webp": "image/webp",
}

func (p *VisionPlugin) resolveImage(path string) (map[string]any, error) {
    // URL → 直接传递
    if strings.HasPrefix(path, "http://") || strings.HasPrefix(path, "https://") {
        return map[string]any{
            "type": "image_url",
            "image_url": map[string]any{
                "url": path,
            },
        }, nil
    }

    // 本地文件 → base64 编码
    ext := strings.ToLower(filepath.Ext(path))
    mimeType, ok := supportedFormats[ext]
    if !ok {
        return nil, fmt.Errorf("unsupported image format: %s (supported: png, jpg, gif, webp)", ext)
    }

    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("cannot read file: %w", err)
    }

    // 检查文件大小（大多数 API 限制 20MB）
    if len(data) > 20*1024*1024 {
        return nil, fmt.Errorf("image too large: %d bytes (max 20MB)", len(data))
    }

    encoded := base64.StdEncoding.EncodeToString(data)
    return map[string]any{
        "type": "image_url",
        "image_url": map[string]any{
            "url": fmt.Sprintf("data:%s;base64,%s", mimeType, encoded),
        },
    }, nil
}
```

### 5. 配置

```toml
[[plugins]]
name = "vision"
enabled = true

[plugins.config]
model = "gpt-4o"                          # 或 "claude-sonnet-4-20250514"
api_key = "secret:sk-xxx"                 # 可独立配置，不影响主模型
api_base = "https://api.openai.com/v1"    # 或 OpenRouter
max_tokens = 1024
```

### 6. 与主模型复用凭据（可选）

如果不配置独立的 `api_key`/`api_base`，可以从主模型配置中继承：

```go
func (p *VisionPlugin) Init(ctx context.Context, config map[string]any) error {
    p.config = parseConfig(config)
    // 如果未独立配置，从主配置继承
    if p.config.APIKey == "" {
        p.config.APIKey = os.Getenv("AI_API_KEY")
    }
    if p.config.APIBase == "" {
        p.config.APIBase = os.Getenv("AI_API_BASE")
        if p.config.APIBase == "" {
            p.config.APIBase = "https://api.openai.com/v1"
        }
    }
    return nil
}
```

## 解决的问题

- **截图分析**：直接从报错截图中提取错误信息，省去用户手动复制
- **UI 审查**：分析渲染后的页面效果，不再局限于 HTML 源码
- **图表理解**：解读监控图表、架构图、流程图中的关键信息
- **OCR 提取**：从图片中提取文字（日志截图、配置截图等）
- **多模态工作流**：与 shell、fsRead 等工具配合（截图 → 分析 → 修复）

## 代价与风险

| 代价 | 评估 |
|------|------|
| **新增插件** | ~300 行 Go，复杂度极低 |
| **API 成本** | 视觉调用比纯文本贵（GPT-4o 约 $0.01/图片），但频率低 |
| **网络依赖** | 需要调用外部 API（与主 LLM 调用相同） |
| **图片大小** | base64 编码后膨胀 33%，需注意大图限制 |

## 可扩展性

- **Phase 2**：集成 `webFetch` 渲染结果 → 截图 → 视觉分析（页面自动审查）
- **Phase 3**：支持多图对比（"对比这两个截图的差异"）
- **Phase 4**：集成 Playwright MCP → 截图 → 视觉分析（完整的浏览器自动化+视觉反馈闭环）

## 参考

- Hermes Agent vision_tools：`tools/vision_tools.py`
- Hermes Agent auxiliary_client：`agent/auxiliary_client.py`
- OpenAI Vision API：`https://platform.openai.com/docs/guides/vision`
- DMR ExtendedTools Hook：`pkg/plugin/hooks.go:14`
- DMR Tool 定义接口：`pkg/tool/tool.go:24-118`
