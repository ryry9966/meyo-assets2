---
title: Git 自动化实践：让 Agent 接管提交信息与分支清理
feedId: 38958
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

用 Agent 干活一段时间后，发现最占时间的往往不是写代码，而是收尾：写提交信息、想分支名、清理合完没删的分支、补变更说明。这些事机械、高频、套路固定，恰好适合交给 AI。OpenClaw 这边的思路是：给 Agent 一套受限的 git 工具（MCP git server 或 shell 包装），配一份固化流程的 skill，让它做"语言层"的工作——描述、命名、归纳；仓库状态的变更权留在人手里。

## 问题

直接让 Agent 操作 git，有三个典型坑：

1. **提交质量差**：只看文件名就编 commit message，diff 根本没读过；
2. **权限过大**：`git add -A` 一把梭，`.env`、日志文件进了暂存区；`push --force`、`reset --hard` 这类破坏性操作没人拦；
3. **分支失控**：任务做完分支留着，三个月后没人敢删。

## 做法

**第一步：权限分层，先只读。** git server 默认只开 `status / diff / log / branch -a` 这类只读命令，跑一两周确认它对仓库的理解没跑偏，再放开 `commit`、`branch`、`merge` 这几个低危写操作。`push`、`reset`、`clean`、`rebase` 一律不进白名单。

**第二步：把提交流程写成 skill。** 核心是固定顺序，不靠模型自由发挥：

```
1. git status 确认工作区状态
2. git diff / git diff --staged 读完整改动
3. git log --oneline -10 学习本仓库的历史风格
4. 起草 conventional commit，说明涉及模块
5. 只 add 相关文件，展示暂存清单
6. 等我确认后再 commit
```

第 3 步很关键——不同仓库的提交习惯差异很大，让 Agent 先看历史，message 风格才不突兀。

**第三步：分支管理任务化。** 两件事：

- 命名约束写进 skill：`feature/<任务号>-短描述`、`fix/...`，Agent 建分支前先自检格式；
- 每周一个定时任务跑"分支审计"：列出已合并未删除、超过 30 天无提交的分支，生成报告推给我。Agent 只产出建议，删除动作由人执行。

## 踩坑记录

- **`git add -A` 是重灾区。** 有一次它把本地调试用的 `.env.local` 一起暂存了，幸好确认环节扫到了。后来在 skill 里明确：add 前必须列出文件清单，且 `.env*`、`*.pem`、`*.log` 属于禁触模式。
- **不读 diff 只看文件名**，生成的 message 会一本正经地错。"必须先读 diff"要写成硬性步骤，不能指望模型自觉。
- **人机并发冲突。** 我本地改着代码，Agent 在另一个会话里同时 commit，状态立刻乱了。现在的约定是：Agent 只在专属 worktree 里操作，主工作区不碰。
- **已合并 ≠ 可以删。** 审计报告里补了一条：含未合并 commit 的分支单独标出，Agent 不得建议删除，只标注"需人工判断"。

## 可复用建议

1. **先只读、后写、永不 push**——权限按周逐步放开，比一次给全安全得多；
2. 允许/禁止的 git 子命令写进 skill 配置文件，别靠 prompt 里一句"小心点"；
3. 所有 Agent 的 git 操作落一份日志，出问题可回溯；
4. 破坏性操作（force push、reset、clean）设为绝对禁区，任务再急也不开例外；
5. 用 conventional commit 约束格式，日志机器可读，后续做 CHANGELOG 自动化有地基。

## 总结

这套流程跑下来，日常提交基本变成"Agent 起草、我扫一眼确认"，分支不再是历史包袱。核心经验一句话：让 AI 处理 git 里"写什么、叫什么"的语言问题，把"动不动仓库状态"的决定权留在人手里。自动化程度可以慢慢加，护栏要第一天就立好。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/6bb26d6991e593be.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/653388fbe75f76ba.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/2a31462972e4d803.png)

