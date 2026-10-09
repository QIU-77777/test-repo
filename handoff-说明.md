# Devin `/handoff` 机制说明

> 本文由云端 Devin session 写成，用于解释 `/handoff` 之后本地与云端之间的关系。
> 文件本身也是演示：它在云端 VM 上创建、commit、push，再由你在本地 `git fetch` 拿回来。

## 1. 一句话结论

`/handoff` 不是把本地会话"搬"到云端，而是 **用本地会话的一份快照，在云端新建一个独立的 session**。
之后两个 session 各自独立运行，互不同步。

```
本地 Mac (Devin CLI)                          云端 VM (Devin Cloud)
┌─────────────────────────┐                   ┌─────────────────────────────┐
│ 本地 session             │   /handoff        │ 云端 session（新建）          │
│ - 对话历史（继续保留）     │ ── 快照 ──────▶   │ - 注入：最近 prompts + 上一轮回复 │
│ - 工作目录 ~/Desktop/prd │                   │ - 注入：repo 标识 + 分支       │
│ - .git → origin          │                   │ - 注入：未提交改动的 diff       │
└─────────────────────────┘                   │ - 工作目录：/home/ubuntu/repos │
            │                                  └─────────────────────────────┘
            │                                                │
            │           git push / git fetch                 │
            └──────────── GitHub (origin) ◀──────────────────┘
```

## 2. 两个 session 的关系

| 维度 | 本地 session（CLI） | 云端 session（本文所在） |
|---|---|---|
| 是否仍存在 | 是，历史留在本机 | 是，新建的，有独立的 session URL |
| 对话历史 | 完整 | 只有 handoff 时刻注入的那份快照 |
| 之后的对话 | 不会出现在云端 | 不会回写到本地 CLI |
| 工作目录 | `/Users/.../Desktop/prd` | `/home/ubuntu/repos/test-repo`（clone 出来的） |
| 文件来源 | 本地磁盘 | `git clone <origin>`，不是从 Mac 上传 |
| 两边同步手段 | 仅 git remote | 仅 git remote |

所以可以把它理解为 **fork**：从某个时间点分叉，之后各走各的。

## 3. handoff 实际带过去了什么

云端 session 第一条消息里能观察到的注入内容：

1. **仓库标识 + 分支**：`QIU-77777/test-repo`、`main`
2. **会话历史快照**：最近若干条用户 prompt + 上一轮 agent 的完整回复（纯文本）
3. **未提交改动的 diff**：附件 `changes_from_local_clone.diff`
   - 对已跟踪的文本文件改动有效，云端可以 `git apply`
   - 二进制文件（如 `.DS_Store`）只有"Binary files differ"一行，无法应用

**没有带过去的：**

- 工作区的完整文件（云端是从 GitHub clone 的，所以 handoff 强制要求有 remote）
- 未跟踪 / 被 `.gitignore` 忽略的文件
- 本地环境变量、凭据、已安装的依赖
- CLI 之外的任何本地状态

## 4. 云端的鉴权方式

云端 clone 用的 remote 是 `https://git-manager.devin.ai/proxy/github.com/QIU-77777/test-repo.git`，
即 Devin 的 git 代理。它持有你在 Devin 后台授权的 GitHub 凭据，所以云端 VM 不需要你的 token，
你本地的 `gh auth login` 也不会影响云端。

## 5. 云端改动怎么回到本地

没有"云端 → 本地文件系统"的直接回传通道，桥梁只有 git：

```
云端：新建分支 → commit → push → 开 PR
本地：git fetch origin
      git checkout <分支名>          # 直接看云端改动
      # 或者合并 PR 后：
      git checkout main && git pull origin main
```

不适合进 git 的产物（截图、报告、二进制），云端会作为会话附件发出，手动下载即可。

## 6. 迁移到其他产品时值得保留的设计点

- **用 git remote 作为唯一的文件同步通道**：省掉双向文件同步的复杂度，天然有版本和冲突处理
- **快照而非共享状态**：handoff 后两端解耦，云端可以长时间独立运行，不依赖本地在线
- **只带 diff 不带工作区**：传输量极小，且不会泄露未跟踪文件（如 `.env`）
- **凭据代理**：云端通过平台代理访问 git，用户凭据不落到 VM 上
- **回传走 PR**：改动必须经过用户审阅后才进入主干，安全边界清晰
