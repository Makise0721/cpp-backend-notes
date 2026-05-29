# 03 — Git 核心操作与工作流

> 关联篇目：[04 测试与 CI/CD](./04-testing-ci-cd.md)

---

## 1. Git 核心概念

```
工作区              暂存区             本地仓库            远程仓库
(Working Dir)    (Staging Area)    (Local Repo)      (Remote Repo)
    │    git add      │   git commit    │    git push     │
    │ ──────────────→ │ ──────────────→ │ ──────────────→ │
    │                 │                 │                 │
    │    git checkout/restore            │    git fetch/pull            │
    │ ←─────────────────────────────── │ ←────────────────────────── │
```

### 1.1 基本操作

```bash
git init                          # 初始化仓库
git clone <url>                   # 克隆
git status                        # 查看状态
git add file.cpp                  # 添加到暂存区
git add -p                        # 交互式选择添加（按 hunk）
git commit -m "message"           # 提交
git commit --amend                # 修改最近一次提交
git log --oneline --graph --all   # 图形化日志
git diff                          # 工作区 vs 暂存区
git diff --staged                 # 暂存区 vs HEAD
```

---

## 2. 分支操作

```bash
git branch                        # 查看本地分支
git branch feature-x              # 创建分支
git checkout feature-x            # 切换分支
git checkout -b feature-x         # 创建 + 切换
git switch -c feature-x           # 同上（Git 2.23+ 推荐）

git merge feature-x               # 合并分支到当前分支
git rebase main                   # 将当前分支变基到 main

git branch -d feature-x           # 删除分支
git push origin --delete feature-x# 删除远程分支
```

### 2.1 Merge vs Rebase

```
Merge:                           Rebase:
main: A---B---C---M              main: A---B---C
         \     /                        \
feat:     D---E                    feat: D'---E'  (提交被重写)

merge: 保留完整历史，有合并提交
rebase: 线性历史，提交更干净
```

| | Merge | Rebase |
|------|------|------|
| 历史 | 保留分叉 | 线性、干净 |
| 冲突解决 | 一次 | 可能需要多次 |
| 公共分支 | ✅ 安全 | ⚠️ 不要 rebase 已推送的分支 |

**黄金规则：不要 rebase 已经推送到远程的公共分支！**

---

## 3. 实用操作

```bash
# cherry-pick: 选择特定提交应用到当前分支
git cherry-pick <commit-hash>
git cherry-pick A..B              # 应用 A 到 B 之间的提交

# stash: 暂存未提交的修改
git stash                         # 暂存
git stash pop                     # 恢复并删除 stash
git stash list                    # 查看所有 stash
git stash apply stash@{1}         # 恢复特定 stash

# reset: 回退（危险！）
git reset --soft HEAD~1           # 撤销 commit，保留修改在暂存区
git reset --mixed HEAD~1          # 撤销 commit + add，保留修改在工作区（默认）
git reset --hard HEAD~1           # 撤销一切，回到指定提交（不可恢复！）

# revert: 安全回退（创建反向提交）
git revert <commit-hash>          # 不影响已有历史

# reflog: 挽救误操作
git reflog                        # 查看 HEAD 移动历史
git reset --hard HEAD@{2}         # 回到 2 步前的状态
```

---

## 4. Git 工作流

### 4.1 Git Flow

```
main ─────●───────●──── (发布)
           \     /
develop ───●──●──●──●── (开发主线)
            \    /  \
feature-1 ───●──●    \  (功能分支)
                      \
release-1.0 ─────●────● (发布分支)
```

| 分支 | 用途 |
|------|------|
| `main` | 生产代码 |
| `develop` | 开发主线 |
| `feature/*` | 功能开发 |
| `release/*` | 发布准备 |
| `hotfix/*` | 紧急修复 |

**适用**：有明确发布周期的项目，版本号管理严格。

### 4.2 GitHub Flow（简化版）

```
main ─────●──●──●──●──● (始终可部署)
           \    \    \
feature-A ──●────    \
feature-B ──────●──── \
feature-C ────────────●
```

- 只有一个长期分支 `main`
- 所有开发在 feature 分支
- 通过 Pull Request 合并回 main
- main 始终可部署

**适用**：持续部署的 Web 服务、小型团队。

### 4.3 Trunk-Based Development

所有开发者直接向 `main`（trunk）提交小步变更。Feature flag 控制未完成功能的可见性。

**适用**：Google/Facebook 等大型科技公司——极高频的集成。

---

## 5. `.gitignore`、`submodule`、`bisect`

### 5.1 `.gitignore`

```gitignore
# 编译产物
build/
*.o
*.exe

# IDE
.vscode/
.idea/

# 依赖
node_modules/

# 例外
!important.o        # 不忽略这个文件
```

### 5.2 `submodule`

```bash
# 添加子模块
git submodule add https://github.com/lib/foo.git third_party/foo

# 克隆带子模块的仓库
git clone --recursive <url>

# 更新子模块
git submodule update --init --recursive
```

### 5.3 `bisect`——二分定位 bug

```bash
git bisect start                  # 开始二分
git bisect bad HEAD               # 当前版本有问题
git bisect good v1.0              # v1.0 是好的

# Git 自动 checkout 到中间某个提交
# 编译 + 测试 → 告诉 Git 结果
git bisect good                   # 这个版本是好的
# 或
git bisect bad                    # 这个版本有问题

# Git 继续二分，直到找到第一个有问题的提交
git bisect reset                  # 结束二分，回到原分支
```

---

## 小结

| 操作 | 用途 |
|------|------|
| `merge` | 合并分支，保留历史 |
| `rebase` | 变基，线性历史，不用于公共分支 |
| `cherry-pick` | 提取特定提交 |
| `stash` | 暂存未提交修改 |
| `reset --soft/hard` | 回退 |
| `reflog` | 救回误删的提交 |
| `bisect` | 二分定位 bug 引入点 |

下一篇 [04 测试与 CI/CD](./04-testing-ci-cd.md)。
