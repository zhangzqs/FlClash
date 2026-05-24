---
name: upgrade-mihomo-kernel
description: Upgrade the mihomo (Clash.Meta) kernel in FlClash to the latest Alpha version. Use when the user wants to sync the kernel, update mihomo, or upgrade Clash.Meta.
---

# Upgrade Mihomo Kernel

将 FlClash 项目中的 mihomo (Clash.Meta) 内核升级到最新 Alpha 版本。

## 仓库关系

```
MetaCubeX/mihomo (上游, Alpha 分支)
    └── zhangzqs/Clash.Meta (用户 fork, FlClash-Alpha 分支)
           └── FlClash/core/Clash.Meta/ (git submodule)
                  └── core/go.mod: replace github.com/metacubex/mihomo => ./Clash.Meta
```

- **Clash.Meta fork**: `git@github.com:zhangzqs/Clash.Meta.git`, 分支 `FlClash-Alpha`
- **上游**: `git@github.com:MetaCubeX/mihomo.git`, 分支 `Alpha`
- **FlClash 定制补丁**: 一个 commit (`feat: support FlClash`)，包含 Android 适配等修改

## 升级流程

### Step 1: 同步 Clash.Meta fork

```bash
# 克隆或进入已有的 Clash.Meta 工作目录
cd /tmp/clash-meta-upgrade
git clone git@github.com:zhangzqs/Clash.Meta.git .
git remote add upstream git@github.com:MetaCubeX/mihomo.git
git fetch upstream Alpha
```

### Step 2: 在最新 Alpha 上应用 FlClash 补丁

```bash
# 基于最新 Alpha 创建新分支
git checkout -b FlClash-Alpha-new upstream/Alpha

# 找到 FlClash 补丁 commit（通常是最新的 "feat: support FlClash"）
# 从原来的 FlClash-Alpha 分支 cherry-pick
git fetch origin FlClash-Alpha
git cherry-pick <flclash-patch-commit>
```

如果有冲突：
- 优先保留 FlClash 补丁的修改（Android 适配等是 FlClash 运行必需的）
- 如果上游 API 变更导致 patch 应用失败，手动调整代码

### Step 3: 推送并更新 submodule

```bash
# 强制更新 FlClash-Alpha 分支
git push --force origin FlClash-Alpha-new:FlClash-Alpha

# 回到 FlClash 仓库
cd /home/zzq/code/repo/zzq/FlClash

# 初始化 submodule（如果尚未初始化）
git submodule update --init --depth 1 core/Clash.Meta

# 更新到最新
cd core/Clash.Meta
git fetch origin FlClash-Alpha
git checkout origin/FlClash-Alpha
cd ../..
```

### Step 4: 更新 Go 依赖

```bash
cd core
go mod tidy
cd ..
```

### Step 5: 修复 API 兼容性问题

新内核可能移除或变更 API。常见的需要检查的地方：

- `core/common.go` — FlClash 核心封装逻辑
- `core/hub.go` — 事件处理和代理管理
- `core/lib.go` — CGO 导出函数
- `core/action.go` — 动作处理
- `core/server.go` — HTTP 服务器

关键 API 变更历史：
- `tunnel.ProxiesWithProviders()` 已被移除 → 使用本地 `getProxiesWithProviders()` 替代
- 注意 `outboundgroup.SelectAble` 接口变更
- 注意 `tunnel.Tunnel` 类型变更

如果 Go 编译报错，逐文件分析上游变更后修复。

### Step 6: 提交并验证

```bash
git add core/Clash.Meta core/go.mod core/go.sum [其他修改的文件]
git commit -m "chore: upgrade mihomo kernel to latest Alpha"
git push origin main
```

### Step 7: CI 验证

```bash
gh workflow run manual-build --repo zhangzqs/FlClash -f platform=android -f arch=arm64
```

等待 CI 通过。如果失败，查看日志修复问题。

## 常用命令

```bash
# 查看当前 submodule commit
git submodule status core/Clash.Meta

# 查看上游 Alpha 最新 commit
gh api repos/MetaCubeX/mihomo/commits/Alpha --jq '{sha: .sha[0:7], date: .commit.committer.date}'

# 查看 Clash.Meta fork FlClash-Alpha 最新 commit
gh api repos/zhangzqs/Clash.Meta/commits/FlClash-Alpha --jq '{sha: .sha[0:7], date: .commit.committer.date}'

# 比较 FlClash-Alpha 落后 Alpha 多少 commit
gh api "repos/zhangzqs/Clash.Meta/compare/Alpha...FlClash-Alpha" --jq '{behind_by: .behind_by, ahead_by: .ahead_by}'
```
