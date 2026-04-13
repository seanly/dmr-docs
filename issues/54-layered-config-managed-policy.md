# P1: 分层配置与管理员策略锁定 — 企业级策略治理

> DMR 当前配置为单层 `config.toml` + OPA 策略文件，缺少配置层级覆盖和管理员锁定机制。Claude Code 支持 user → project → local → managed policy 四级配置，且 managed settings 中的 deny 规则不可被下级覆盖。企业场景需要管理员能统一下发不可篡改的安全基线。

## 问题

### 1. 单层配置无法满足多场景需求

当前 DMR 配置只有一个 `~/.dmr/config.toml`，导致：

- **个人偏好与项目约定混在一起**：用户的全局 model 偏好和某个项目的特殊 OPA 策略在同一个文件
- **无法做项目级配置共享**：团队成员无法通过仓库共享项目的 OPA 策略和插件配置
- **无法做组织级强制策略**：安全团队无法下发 "全公司禁止 `rm -rf /`" 这样的基线规则

### 2. 缺少管理员锁定机制

即使通过 HTTPS URL 分发 OPA 策略，用户仍可以：

- 在本地 `config.toml` 中覆盖 `policies` 列表，跳过远程策略
- 在本地添加 `permissive-policy.rego` 覆盖默认 deny 规则
- 设置 `allow_rules` 白名单绕过策略检查

没有机制保证管理员策略的不可篡改性。

### 3. OPA 策略与 allow_rules 的优先级不透明

当前 allow_rules 是 OPA 评估之前的快速通道（`cache.go` 中的 `AllowRulesCache`），匹配到 allow_rules 的调用直接放行，不经过 OPA 评估。这意味着一条宽泛的 allow_rule（如 `shell:*`）可以绕过所有 OPA deny 规则。

## 设计方案

### 1. 四级配置层次

```text
优先级（高 → 低）：

managed/    — 管理员策略（不可被下级覆盖的 deny 规则）
  └── ~/.dmr/managed/config.toml
  └── ~/.dmr/managed/policies/*.rego

user/       — 用户全局配置
  └── ~/.dmr/config.toml
  └── ~/.dmr/etc/policies/*.rego

project/    — 项目级配置（可提交到仓库）
  └── .dmr/config.toml
  └── .dmr/policies/*.rego

local/      — 项目本地配置（gitignore）
  └── .dmr/config.local.toml
  └── .dmr/policies.local/*.rego
```

### 2. 配置合并规则

```go
// pkg/config/layered.go

type LayeredConfig struct {
    Managed  *Config // 管理员配置：deny 规则不可覆盖
    User     *Config // 用户全局配置
    Project  *Config // 项目级配置
    Local    *Config // 项目本地配置
}

func (lc *LayeredConfig) Merge() *Config {
    // 1. 从 local 开始，逐级向上合并
    merged := lc.Local.mergeWith(lc.Project).mergeWith(lc.User)

    // 2. Managed 的 deny 规则强制追加，不可被下级移除
    merged.ManagedDenyRules = lc.Managed.DenyRules

    // 3. Managed 的锁定标志
    if lc.Managed.AllowManagedPoliciesOnly {
        merged.Policies = lc.Managed.Policies  // 忽略下级的策略配置
    }
    if lc.Managed.AllowManagedAllowRulesOnly {
        merged.AllowRules = lc.Managed.AllowRules  // 忽略下级的 allow_rules
    }

    return merged
}
```

### 3. 管理员锁定配置项

```toml
# ~/.dmr/managed/config.toml

[managed]
# 只加载 managed 目录下的 OPA 策略，忽略用户/项目策略
allow_managed_policies_only = false

# 只使用 managed 目录下的 allow_rules，忽略用户/项目 allow_rules
allow_managed_allow_rules_only = false

# 禁止用户配置的插件（只允许 managed 中明确启用的插件）
allow_managed_plugins_only = false

[managed.plugins.opa_policy]
# 管理员策略目录（这些策略始终加载，用户无法跳过）
policies = ["~/.dmr/managed/policies/"]
```

### 4. 管理员策略分发

管理员策略通过以下方式投放到 `~/.dmr/managed/`：

- **MDM (macOS)**：通过 Jamf/Mosyle 等 MDM 工具推送配置 profile
- **包管理器**：通过 deb/rpm 包安装到 `/etc/dmr/managed/`，symlink 到 `~/.dmr/managed/`
- **启动时拉取**：在 `config.toml` 中配置 managed 策略的 HTTPS URL，启动时自动同步

```go
// pkg/config/managed.go

func SyncManagedPolicies(cfg *Config) error {
    for _, url := range cfg.Managed.PolicyURLs {
        // 从 HTTPS URL 下载策略到 ~/.dmr/managed/policies/
        // 校验签名（可选）
        // 写入本地并设置只读权限
    }
    return nil
}
```

### 5. OPA 评估中的层级感知

```rego
# managed policy 中的 deny 规则示例

# 管理员基线：禁止任何形式的递归删除
decision := {
    "action": "deny",
    "reason": "[managed] recursive deletion is prohibited by organization policy",
    "risk": "critical"
} if {
    input.tool == "shell"
    regex.match(`\brm\b.*-[a-zA-Z]*r`, input.args.command)
}
```

managed 策略的 deny 规则在 OPA 引擎中优先级最高，用户策略中的 allow 规则无法覆盖。

### 6. 项目级配置共享

```text
my-project/
  .dmr/
    config.toml          # 项目级配置（提交到仓库）
    policies/
      project.rego       # 项目特有的策略规则
    .gitignore           # 包含 config.local.toml 和 policies.local/
```

团队成员克隆仓库后，项目级策略自动生效。个人偏好放在 `config.local.toml` 中。

## 实现步骤

- [ ] 定义 `LayeredConfig` 和合并逻辑（`pkg/config/layered.go`）
- [ ] 修改 `LoadDefault()` 支持多级配置加载
- [ ] 实现 `~/.dmr/managed/` 目录扫描和配置加载
- [ ] 在 OPA 引擎中区分 managed / user / project 策略的优先级
- [ ] 实现 `allow_managed_policies_only` 等锁定标志
- [ ] 支持 `.dmr/config.toml` 项目级配置（检测 workspace 目录）
- [ ] 修改 `dmr config show` 显示配置来源（哪一层生效）
- [ ] 添加 `dmr config layers` 子命令展示配置层级
- [ ] 文档：管理员部署指南
- [ ] 测试：多级配置合并、锁定标志、deny 不可覆盖

## 代价与风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| 配置合并逻辑复杂，debug 困难 | 中 | 中 | `dmr config show --verbose` 显示每个值的来源层级 |
| 项目级 `.dmr/config.toml` 被恶意仓库利用 | 中 | 高 | 项目级配置不能放松 managed/user 级的 deny 规则；首次加载时提示用户确认 |
| managed 目录权限问题 | 低 | 中 | 启动时校验 managed 目录权限；非 root 用户无法修改 |
| 向后兼容：现有单层配置的用户 | 确定 | 低 | 单层配置等价于 user 级配置，无需迁移 |

## 参考

- Claude Code 配置层级：`~/.claude/settings.json` → `.claude/settings.json` → `.claude/settings.local.json` → managed policy
- Claude Code `allowManagedHooksOnly` 和 `allowManagedPermissionRulesOnly`
- `pkg/config/config.go` — 当前配置加载逻辑
- `plugins/opapolicy/engine_build.go` — 策略源加载
- `plugins/opapolicy/allow_rules.go` — allow_rules 快速通道
