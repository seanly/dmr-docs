# Git Worktree 开发流程指南

## 什么是 Worktree

Git Worktree 允许从同一个仓库同时检出多个工作目录，每个目录对应不同的分支。多个 worktree **共享同一个 `.git` 仓库**，但拥有独立的工作区、索引和编译缓存。

```
project/
├── dmr/                    # main 分支工作区（稳定）
├── dmr-feature-x/          # feature-x 分支工作区（开发中）
├── dmr-hotfix/             # hotfix 分支工作区（紧急修复）
└── .git/                   # 共享的 git 仓库（只存一份）
```

## 为什么用 Worktree

传统分支切换的痛点：

```bash
git checkout -b feature-x     # 从 main 切出
# ... 大量修改，编译缓存建立 ...
git checkout main              # 切回去修 Bug → 编译缓存失效
git checkout feature-x         # 切回来 → 工作区重建
```

| 对比 | 分支切换 | Worktree |
|------|---------|----------|
| 同时工作分支数 | 1 个 | 多个 |
| 切换成本 | 重建工作区 + 编译缓存失效 | cd 切换目录，零成本 |
| 适用场景 | 小改动 | 大重构 + 长期并行开发 |

## 基础操作

### 创建 Worktree

```bash
# 进入主仓库
cd ~/projects/dmr

# 从当前 HEAD 创建新 worktree + 新分支
git worktree add ../dmr-feature-x -b feature-x

# 从指定分支/tag 创建
git worktree add ../dmr-hotfix -b hotfix-123 main

# 查看所有 worktree
git worktree list
# /home/user/projects/dmr             main         [abc1234]
# /home/user/projects/dmr-feature-x   feature-x    [abc1234]
```

### 在 Worktree 中工作

```bash
# 直接 cd 进入，就是一个完整的工作区
cd ../dmr-feature-x

# 正常的 git 操作全部可用
git status
git add -A
git commit -m "feat: add new feature"
git push -u origin feature-x
```

### 删除 Worktree

```bash
# 回到主仓库
cd ~/projects/dmr

# 删除 worktree（工作区干净时）
git worktree remove ../dmr-feature-x

# 强制删除（有未提交更改时）
git worktree remove -f ../dmr-feature-x

# 清理已失效的 worktree 记录
git worktree prune
```

## 典型开发流程

### 场景一：大型重构

适用于插件接口重构、架构调整等跨多文件的大改动。

```bash
# 1. 创建重构专用 worktree
cd ~/projects/dmr
git worktree add ../dmr-refactor -b refactor/plugin-v2

# 2. 在新 worktree 中开发（不影响主分支）
cd ../dmr-refactor
# ... 编写代码、编译、测试 ...
go build ./cmd/dmr/
go test ./pkg/plugin/...

# 3. 期间主分支需要修 Bug？直接切到主 worktree
cd ../dmr
# ... 修复 Bug、提交、推送 ...
git commit -am "fix: critical bug"
git push

# 4. 回到重构 worktree 继续工作（编译缓存仍在）
cd ../dmr-refactor
# 继续开发，零切换成本

# 5. 同步主分支更新
git fetch origin main:main
git merge main
# 解决冲突（如有）

# 6. 完成后推送、创建 PR
git push -u origin refactor/plugin-v2
gh pr create --title "refactor: plugin interface v2"

# 7. 合并后清理
cd ../dmr
git worktree remove ../dmr-refactor
git branch -d refactor/plugin-v2
```

### 场景二：多功能并行开发

同时推进多个独立功能。

```bash
cd ~/projects/dmr

# 创建多个 worktree
git worktree add ../dmr-sandbox -b feature/sandbox
git worktree add ../dmr-fts -b feature/fts-search
git worktree add ../dmr-hotfix -b hotfix/issue-42

# 多终端/多 IDE 窗口并行工作
# Terminal 1: cd ../dmr-sandbox
# Terminal 2: cd ../dmr-fts
# Terminal 3: cd ../dmr-hotfix

# 查看全局状态
git worktree list
```

### 场景三：对比调试

两个 worktree 打开同一文件的不同版本进行对比。

```bash
# 窗口 1: 旧版本
code ~/projects/dmr

# 窗口 2: 新版本
code ~/projects/dmr-refactor

# 两个 VS Code 窗口完全独立
# 可以同时运行测试、断点调试、对比行为差异
```

## 与主分支同步

### 拉取主分支更新

```bash
cd ~/projects/dmr-feature-x

# 方法 1: merge（保留完整提交历史）
git fetch origin main:main
git merge main

# 方法 2: rebase（线性历史，推荐短生命周期分支）
git fetch origin main
git rebase origin/main
```

### 主分支紧急修复

```bash
# 不需要 stash，不需要切分支，直接 cd
cd ~/projects/dmr         # 主 worktree
git pull
# ... 修复并推送 ...

cd ~/projects/dmr-feature-x   # 回到功能 worktree
# 你的改动原封不动
```

## 与 IDE 集成

### VS Code

每个 worktree 是独立目录，用 `code <path>` 打开即可。每个窗口有独立的：
- 扩展设置
- 终端会话
- 调试配置
- 编辑器状态

```bash
code ~/projects/dmr              # 窗口 1: main
code ~/projects/dmr-feature-x    # 窗口 2: feature-x
```

### JetBrains IDE

File → Open → 选择 worktree 目录，以独立项目打开。

## 与 CI/CD 集成

```yaml
# .github/workflows/test-feature.yml
name: Test Feature Branch

on:
  push:
    branches: [feature/*, refactor/*]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build and Test
        run: |
          go build ./cmd/dmr/
          go test ./...
```

## 命令速查

```bash
# ===== 创建 =====
git worktree add <path> -b <branch>              # 创建 worktree + 新分支
git worktree add <path> -b <branch> <base>        # 指定起点
git worktree add <path> <existing-branch>          # 使用已有分支

# ===== 查看 =====
git worktree list                                  # 列出所有 worktree
git worktree list --porcelain                      # 机器可读格式

# ===== 同步 =====
git fetch origin main:main                         # 更新 main 引用
git merge main                                     # 合并到当前分支
git rebase origin/main                             # 变基到 main

# ===== 清理 =====
git worktree remove <path>                         # 删除 worktree
git worktree remove -f <path>                      # 强制删除
git worktree prune                                 # 清理失效记录

# ===== 高级 =====
git worktree lock <path>                           # 锁定防止误删
git worktree unlock <path>                         # 解锁
git worktree move <old-path> <new-path>            # 移动位置
```

## 最佳实践

| 实践 | 说明 |
|------|------|
| **命名规范** | worktree 目录用 `<项目>-<功能>` 格式，如 `dmr-sandbox` |
| **及时清理** | PR 合并后立即 `git worktree remove`，避免目录堆积 |
| **独立编译** | 每个 worktree 有独立编译缓存，不要共享 `build/` 或 `node_modules/` |
| **配置隔离** | 不同 worktree 使用不同配置文件，避免冲突 |
| **提交粒度** | worktree 中的提交保持独立、清晰，方便 review |
| **锁定长期 worktree** | 长期保留的用 `git worktree lock` 防止误删 |
| **限制数量** | 同时活跃的 worktree 不超过 3-4 个，否则难以管理 |

## 注意事项

- 同一个分支不能被两个 worktree 同时检出
- 所有 worktree 共享 remote、stash、reflog
- `git worktree remove` 不会删除分支，只删除工作目录
- worktree 中的 `git log`、`git branch` 等命令看到的是同一个仓库
- 子模块（submodule）需要在每个 worktree 中单独 `git submodule update`
