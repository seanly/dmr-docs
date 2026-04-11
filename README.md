# DMR Docs

DMR 项目的设计讨论与方案文档仓库。

这里不存放代码，只存放**想法、提案和技术方案**。所有对 DMR 架构、功能、缺陷的分析和改进建议都在这里讨论，经过讨论确认后再到 [seanly/dmr](https://github.com/seanly/dmr) 中实施。

## 目录结构

```
dmr-docs/
├── issues/          # 设计提案与问题讨论
├── docs/            # 开发指南与参考文档
└── AGENTS.md        # issues/ 目录的文档格式规范
```

## Issues 概览

`issues/` 目录中的文档同步维护为 [seanly/dmr](https://github.com/seanly/dmr/issues) 的 GitHub Issues，按三大类组织：

### 值得借鉴的能力

从 [TapeAgents](https://github.com/ServiceNow/TapeAgents) 框架对比分析后，识别出的 DMR 可借鉴能力。

| # | 优先级 | 主题 |
|---|--------|------|
| 00 | P0 | 确定性重放测试 |
| 01 | P1 | LLM 调用观测与审计 |
| 02 | P1 | Tape 谱系追踪 |
| 03 | P2 | 分级观察裁剪 |
| 04 | P2 | LLM 响应缓存 |
| 05 | P3 | Entry Schema 验证 |
| 06 | P3 | Subagent 结构化返回 |
| 07 | P3 | LLM 输出解析失败重试 |
| 08 | P2 | 多 Agent 编排模式 |

### 设计缺陷

通过与 TapeAgents 的架构对比，暴露出的 DMR 结构性问题。

| # | 优先级 | 主题 |
|---|--------|------|
| 09 | P0 | TapeStore.Append 不返回错误 |
| 10 | P0 | ToolContext 并发数据竞争 |
| 11 | P1 | Hook 系统无 panic 恢复 + 切片竞争 |
| 12 | P1 | 三重 Compact 缺乏协调 |
| 13 | P1 | 工具执行无超时无取消 |
| 14 | P1 | SQLite Store 读写锁误用 |
| 15 | P2 | 错误分类过宽导致错误 Compact |
| 16 | P2 | HTTP 5xx 不重试 + 流式中断不可恢复 |
| 17 | P2 | thinkFilter O(n^2) + 无界缓冲区 |

### 未来方向

深入分析后识别出的 DMR 可发展方向。

| # | 优先级 | 主题 |
|---|--------|------|
| 18 | P2 | Tape 数据挖掘与 Prompt 优化 |
| 19 | P2 | 交互式调试 UI |
| 20 | P1 | 可解释动作 |
| 21 | P2 | Plan-Survey-Act 结构化推理 |
| 22 | P1 | 隔离式工具执行 |
| 23 | P1 | 动作重复检测与集成投票 |
| 24 | P3 | Tape 浏览器与人工审查标注 |
| 25 | P0 | 自主进化架构 |
| 25a | — | 自主进化架构批判性分析 |
| 26 | P1 | 集成投票标注 |
| 27 | P1 | Tape 增强提案 |
| 28 | P1 | Sandbox 集成 |
| 29 | P1 | 插件接口重构 |

## 如何贡献

1. 新建 `issues/{编号}-{主题}.md`，遵循 [AGENTS.md](AGENTS.md) 中的格式规范
2. 提交 PR 讨论
3. 确认后同步创建到 [seanly/dmr Issues](https://github.com/seanly/dmr/issues)
