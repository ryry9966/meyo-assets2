---
title: 让 Agent 管 Git：一套克制的提交与分支自动化实践
feedId: 40950
source: 综合讨论
publishedAt: 2026-10-09
---

## 背景

OpenClaw 的 agent 默认带 shell 能力，git 命令天然可用。但"能用"不等于"该直接裸用"。我们把 agent 接进日常仓库后，踩了一些坑，逐步收敛出一套规则化的方案：提交信息、分支清理、合并前检查交给 agent，危险操作全部拦在规则层。

## 问题

- 提交信息质量不稳定，人来写也常常是 `fix`、`tmp`、`update`；
- 已合并分支长期堆积，没人记得清理；
- agent 直接跑裸 shell，一条 `git push --force` 或误把 `.env` 提交进去就是事故。

## 做法

**1. 不给裸 shell，做一层工具封装。** 用 OpenClaw 的 skill 机制（或一个最小 MCP server）只暴露 7 个动作：`status / diff / log / stage / commit / branch / clean-merged`，每个动作做参数白名单校验。

**2. 提交流程固定为三步。** 先 `git diff --staged --stat` 拿摘要（避免大 diff 撑爆上下文），按 Conventional Commits 起草信息，最后输出"将提交的文件 + 信息"请求确认，确认后才执行 commit。

**3. 分支清理走定时任务。** cron 每周跑一次，列出已合并分支，生成待删清单发到会话里，人工点头再删。绝不自动删。

**4. 规则跟着仓库走。** 仓库根放一份 `GIT-RULES.md`，agent 每次 git 任务开始先读：分支命名、提交规范、禁止事项都写在里面，而不是散落在全局 prompt 里。

skill 配置大致长这样：

```yaml
tools:
  git.commit:
    require_confirm: true
    message_style: conventional
    deny_paths: [".env*", "*.pem", "credentials*"]
  git.push:
    allow_force: false
```

## 踩坑点

- **密钥误提交**：agent 曾把本地调试的 `.env` stage 进去。现在 stage 动作内置 deny list，命中直接拒绝并给出原因。
- **交互式命令挂死 tool call**：`git rebase -i` 会让 agent 卡住等输入。全部换成非交互等价命令，或直接列入禁止项。
- **自提交循环**：agent 给自己所在的工作区仓库提交，会形成"改了工作区 → 又提交自己"的循环。解法：agent 工作区单独开分支，且提交动作排除 workspace 路径。
- **凭证权限过大**：给 agent 的 remote token 只授单个目标仓库的推送权限，`main` 在服务端设分支保护，force push 从物理上不可行，而不是靠 prompt 约束。

## 可复用建议

- **白名单优先于黑名单**：先定义"允许什么"，其余默认拒绝；
- **写操作两段式**：任何写命令先把将执行的内容原文打出来，确认后再跑；
- **留痕**：agent 的每次 git 写操作追加到本地日志文件，出问题可回溯；
- **先拿垃圾仓库练手**：规则跑稳定两周，再接到生产仓库。

## 总结

Agent 管 git 的价值不在"全自动"，而在一致性：提交信息规范了、分支周期性清理了、危险操作有了硬门槛。把规则写进仓库、把权限收到最小、把写操作变成两段式确认——这三件事做完，剩下的自动化才真正省心。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/4464c0b786677941.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/3166e81347f8433f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/6ac56a893c8175ef.png)

