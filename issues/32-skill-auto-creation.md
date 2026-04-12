# P1: 技能自创建 — Agent 从成功任务中自动提取可复用技能

> 灵感来源：Hermes Agent `skill_manage(create/edit/patch)` + 安全扫描 + SKILL.md frontmatter 规范
>
> DMR 现有 skill 插件只能读取和激活技能，无法创建。这限制了 Agent 从实践中学习的能力。

## 问题

DMR 的 skill 插件（`plugins/skill/skill.go`）当前只有**读取**能力：

```go
// plugins/skill/skill.go:71-90 — 当前唯一的工具
func (p *SkillPlugin) skillTool() *tool.Tool {
    return &tool.Tool{
        Name:        "skill",
        Description: "Load a skill by name",
        Parameters:  map[string]any{...},  // 只有 name 参数
        Handler:     p.skillHandler,       // 只读取 SKILL.md 内容
    }
}
```

| 缺失 | 影响 |
|------|------|
| **无法创建技能** | Agent 完成复杂任务后，经验无法沉淀 |
| **无法编辑技能** | 发现技能步骤过时或错误时，无法自行修正 |
| **无安全扫描** | 用户手动放入的 SKILL.md 未经校验 |
| **无 frontmatter 验证** | 格式不规范的技能会导致加载异常 |

## Hermes Agent 的做法

### 1. 技能创建

```python
# hermes_agent/tools/skill_manager_tool.py:293-347

def skill_manage(action='create', name, content, category):
    # 1. 校验名称（小写 + 连字符，≤64 字符）
    # 2. 校验 frontmatter（YAML 格式、必需字段 name/description）
    # 3. 跨所有技能目录检查名称冲突
    # 4. 创建目录 ~/.hermes/skills/[category]/name/
    # 5. 写入 SKILL.md（原子写入：temp + os.replace）
    # 6. 安全扫描（shell 注入、凭据泄露、可疑 import）
    # 7. 扫描失败 → 回滚（删除目录）
```

### 2. 技能修补（模糊匹配）

```python
# skill_manager_tool.py:383-468

def skill_manage(action='patch', name, old_string, new_string):
    # 使用 fuzzy_match 引擎（处理空白差异、缩进、转义）
    # 修补后重新校验 frontmatter
    # 重新安全扫描，失败则回滚
```

### 3. 安全扫描

```python
# tools/skills_guard.py (38KB+)

# 扫描内容：
# - Shell 元字符在未引用的命令字符串中
# - 凭据泄露模式（API key、token）
# - 可疑 Python import（subprocess、urllib、socket）
# - 文件写入模式（超出允许子目录）
```

### 4. 技能引导

```python
# 工具描述中内嵌的创建指导：
# "Create when: complex task succeeded (5+ calls),
#  errors overcome, user-corrected approach worked"
# "Good skills: trigger conditions, numbered steps,
#  exact commands, pitfalls section, verification steps"
```

## DMR 的实现方案

### 1. 扩展现有 skill 插件

在 `plugins/skill/skill.go` 中新增 `skillCreate` 和 `skillEdit` 工具。

```go
// plugins/skill/skill.go — 扩展 registerTools

func (p *SkillPlugin) registerTools(ctx context.Context, args ...any) (any, error) {
    return []*tool.Tool{
        p.skillTool(),      // 现有：读取
        p.skillCreateTool(), // 新增：创建
        p.skillEditTool(),   // 新增：编辑/修补
        p.skillDeleteTool(), // 新增：删除
        p.skillListTool(),   // 新增：列出所有技能
    }, nil
}
```

### 2. 技能创建

```go
// plugins/skill/create.go

func (p *SkillPlugin) skillCreateTool() *tool.Tool {
    return &tool.Tool{
        Name: "skillCreate",
        Description: `Create a new reusable skill from a successful task.

Create when:
- Complex task succeeded (5+ tool calls)
- You overcame errors through trial and learned a pattern
- User corrected your approach and the corrected version worked

Good skills include:
- Trigger conditions (when to use)
- Numbered steps with exact commands
- Pitfalls and common errors section
- Verification steps`,
        Parameters: map[string]any{
            "type": "object",
            "properties": map[string]any{
                "name": map[string]any{
                    "type":        "string",
                    "description": "Skill name (lowercase, hyphens, max 64 chars). e.g. 'deploy-k8s-cluster'",
                    "pattern":     "^[a-z0-9][a-z0-9-_]{0,63}$",
                },
                "description": map[string]any{
                    "type":        "string",
                    "description": "One-line description (max 256 chars)",
                    "maxLength":   256,
                },
                "content": map[string]any{
                    "type":        "string",
                    "description": "Full SKILL.md content with YAML frontmatter and markdown body",
                },
                "category": map[string]any{
                    "type":        "string",
                    "description": "Category subdirectory (e.g. 'devops', 'coding'). Default: 'general'",
                },
            },
            "required": []string{"name", "description", "content"},
        },
        Handler:    p.createHandler,
        Group:      tool.ToolGroupCore,
        SearchHint: "skill, create, save, learn, remember procedure, 技能, 创建",
    }
}

func (p *SkillPlugin) createHandler(ctx *tool.ToolContext, args map[string]any) (any, error) {
    name, _ := args["name"].(string)
    description, _ := args["description"].(string)
    content, _ := args["content"].(string)
    category, _ := args["category"].(string)
    if category == "" {
        category = "general"
    }

    // 1. 名称校验
    if !isValidSkillName(name) {
        return map[string]any{
            "error": "invalid name: must be lowercase, hyphens/underscores, 1-64 chars",
        }, nil
    }

    // 2. frontmatter 校验
    if err := validateFrontmatter(content, name, description); err != nil {
        return map[string]any{"error": fmt.Sprintf("invalid frontmatter: %v", err)}, nil
    }

    // 3. 名称冲突检测
    if existing := p.findSkill(name); existing != nil {
        return map[string]any{
            "error": fmt.Sprintf("skill %q already exists at %s", name, existing.Location),
        }, nil
    }

    // 4. 创建目录
    skillDir := filepath.Join(p.roots[0], category, name)
    if err := os.MkdirAll(skillDir, 0755); err != nil {
        return map[string]any{"error": fmt.Sprintf("mkdir failed: %v", err)}, nil
    }

    // 5. 写入 SKILL.md（原子写入）
    skillPath := filepath.Join(skillDir, "SKILL.md")
    tmpPath := skillPath + ".tmp"
    if err := os.WriteFile(tmpPath, []byte(content), 0644); err != nil {
        os.RemoveAll(skillDir)
        return map[string]any{"error": fmt.Sprintf("write failed: %v", err)}, nil
    }

    // 6. 安全扫描
    if err := scanSkillContent(content); err != nil {
        os.RemoveAll(skillDir) // 回滚
        return map[string]any{"error": fmt.Sprintf("security scan failed: %v", err)}, nil
    }

    // 7. 提交
    if err := os.Rename(tmpPath, skillPath); err != nil {
        os.RemoveAll(skillDir)
        return map[string]any{"error": fmt.Sprintf("commit failed: %v", err)}, nil
    }

    // 8. 刷新缓存
    p.refreshSkills()

    return map[string]any{
        "success":  true,
        "name":     name,
        "location": skillDir,
        "hint":     "Skill created. You can add supporting files (templates, scripts) to " + skillDir,
    }, nil
}
```

### 3. Frontmatter 校验

```go
// plugins/skill/validate.go

func validateFrontmatter(content, expectedName, expectedDesc string) error {
    if !strings.HasPrefix(strings.TrimSpace(content), "---") {
        return fmt.Errorf("must start with YAML frontmatter (---)")
    }

    parts := strings.SplitN(content, "---", 3)
    if len(parts) < 3 {
        return fmt.Errorf("missing closing --- for frontmatter")
    }

    var fm map[string]any
    if err := yaml.Unmarshal([]byte(parts[1]), &fm); err != nil {
        return fmt.Errorf("invalid YAML: %w", err)
    }

    if fm["name"] == nil || fmt.Sprint(fm["name"]) == "" {
        return fmt.Errorf("frontmatter must have 'name' field")
    }
    if fm["description"] == nil || fmt.Sprint(fm["description"]) == "" {
        return fmt.Errorf("frontmatter must have 'description' field")
    }

    body := strings.TrimSpace(parts[2])
    if body == "" {
        return fmt.Errorf("skill body (after frontmatter) cannot be empty")
    }

    return nil
}
```

### 4. 安全扫描

```go
// plugins/skill/security.go

var skillSecurityPatterns = []struct {
    pattern *regexp.Regexp
    reason  string
}{
    {regexp.MustCompile(`(?i)rm\s+-rf\s+/`), "destructive command: rm -rf /"},
    {regexp.MustCompile(`(?i)curl\s+.*\|\s*sh`), "pipe to shell execution"},
    {regexp.MustCompile(`(?i)\$\{?\w*(API_KEY|TOKEN|SECRET|PASSWORD)`), "credential reference"},
    {regexp.MustCompile(`(?i)eval\s*\(`), "eval() usage"},
    {regexp.MustCompile(`(?i)exec\s*\(`), "exec() usage"},
    {regexp.MustCompile(`(?i)base64\s+-d.*\|\s*(sh|bash)`), "base64 decode to shell"},
}

func scanSkillContent(content string) error {
    for _, sp := range skillSecurityPatterns {
        if sp.pattern.MatchString(content) {
            return fmt.Errorf("blocked: %s", sp.reason)
        }
    }
    return nil
}
```

### 5. 技能编辑（修补）

```go
// plugins/skill/edit.go

func (p *SkillPlugin) skillEditTool() *tool.Tool {
    return &tool.Tool{
        Name: "skillEdit",
        Description: "Edit an existing skill's SKILL.md content by replacing a substring",
        Parameters: map[string]any{
            "type": "object",
            "properties": map[string]any{
                "name":       map[string]any{"type": "string", "description": "Skill name"},
                "old_string": map[string]any{"type": "string", "description": "Text to find"},
                "new_string": map[string]any{"type": "string", "description": "Replacement text"},
            },
            "required": []string{"name", "old_string", "new_string"},
        },
        Handler: p.editHandler,
        Group:   tool.ToolGroupCore,
    }
}

func (p *SkillPlugin) editHandler(ctx *tool.ToolContext, args map[string]any) (any, error) {
    name, _ := args["name"].(string)
    oldStr, _ := args["old_string"].(string)
    newStr, _ := args["new_string"].(string)

    sk := p.findSkill(name)
    if sk == nil {
        return map[string]any{"error": fmt.Sprintf("skill %q not found", name)}, nil
    }

    path := filepath.Join(sk.Location, "SKILL.md")
    data, err := os.ReadFile(path)
    if err != nil {
        return map[string]any{"error": err.Error()}, nil
    }

    content := string(data)
    count := strings.Count(content, oldStr)
    if count == 0 {
        return map[string]any{"error": "old_string not found in SKILL.md"}, nil
    }
    if count > 1 {
        return map[string]any{"error": fmt.Sprintf("old_string matches %d times, must be unique", count)}, nil
    }

    newContent := strings.Replace(content, oldStr, newStr, 1)

    // 重新校验 frontmatter
    if err := validateFrontmatter(newContent, name, ""); err != nil {
        return map[string]any{"error": fmt.Sprintf("edit broke frontmatter: %v", err)}, nil
    }

    // 重新安全扫描
    if err := scanSkillContent(newContent); err != nil {
        return map[string]any{"error": fmt.Sprintf("security scan failed after edit: %v", err)}, nil
    }

    // 原子写入
    tmpPath := path + ".tmp"
    if err := os.WriteFile(tmpPath, []byte(newContent), 0644); err != nil {
        return map[string]any{"error": err.Error()}, nil
    }
    if err := os.Rename(tmpPath, path); err != nil {
        return map[string]any{"error": err.Error()}, nil
    }

    p.refreshSkills()

    return map[string]any{
        "success": true,
        "name":    name,
    }, nil
}
```

### 6. 配置

```toml
[[plugins]]
name = "skill"
enabled = true

[plugins.config]
roots = ["~/.dmr/skills/local"]
# 新增配置
allow_create = true               # 允许 Agent 创建技能
security_scan = true              # 创建/编辑时执行安全扫描
max_skill_size = 65536            # SKILL.md 最大字节数
```

## 解决的问题

- **经验沉淀**：复杂任务完成后，关键步骤自动保存为可复用技能
- **知识传递**：团队成员的操作经验以技能形式共享
- **自我修正**：发现技能步骤过时或错误时，Agent 可自行修正
- **安全保障**：所有创建和编辑操作都经过安全扫描，防止注入

## 代价与风险

| 代价 | 评估 |
|------|------|
| **新增代码** | ~400 行 Go，在现有 skill 插件上扩展 |
| **磁盘空间** | 每个技能 \<10KB，可忽略 |
| **安全风险** | Agent 生成的技能可能包含不安全命令 → 通过安全扫描缓解 |
| **质量控制** | Agent 可能创建低质量技能 → 通过 frontmatter 校验 + 后台复查缓解 |

## 与其他 Issue 的关系

- **依赖 #33 后台自学习**：自学习机制负责触发技能创建的时机，skill 插件负责执行
- **互补 #30 记忆系统**：记忆存储"事实和偏好"，技能存储"操作流程"
- **为 #25 自主进化提供数据**：高频使用的技能可作为 Tier 0 模板模式的候选

## 参考

- Hermes Agent skill_manage：`tools/skill_manager_tool.py:293-468`
- Hermes Agent skills_guard：`tools/skills_guard.py`
- Hermes Agent frontmatter 规范：`tools/skill_manager_tool.py:138-174`
- DMR 现有 skill 插件：`plugins/skill/skill.go:20-144`
- DMR SKILL.md 解析：`plugins/skill/skill.go:230-276`
