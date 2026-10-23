---
title: 让 OpenClaw 帮你管 Git：提交与分支自动化的稳妥做法
feedId: 42338
source: 综合讨论
publishedAt: 2026-10-23
---

## 背景

OpenClaw 的 agent 本身就能执行本地命令，git 是最容易被"顺手自动化"的场景。我把自己常用仓库的提交和分支维护交给 agent 跑了一个多月，结论是：值得做，但前提是把权限和流程收紧，而不是丢一句"帮我把代码提交了"。

## 问题

日常开发的痛点其实很碎：

- 改完代码懒得写 commit message，历史里全是 "update"、"fix"；
- 分支用完不删，一个月后 `git branch` 列出十几条早已合并的死分支；
- 忘了哪些改动没提交，切分支时才发现工作区是脏的。

这些事单独看都不大，但高频、重复、有固定套路，正适合外包给 agent。

## 做法

我的方案分三层，全部通过一个自定义 skill 实现：

**1. 命令白名单。** skill 里只放行只读和低危命令：`status`、`diff`、`log`、`branch -vv`、`add`、`commit`。`push`、`reset --hard`、`branch -D`、`rebase` 一律进入"需确认"队列，由我在对话里手动放行。

**2. 提交流程模板化。** 固定步骤：agent 先跑 `git status` 和 staged diff，按 Conventional Commits 生成 2~3 条候选 message，附上它对这次改动的一句话理解，我确认后才执行 `git add <指定文件>` + `commit`。关键点：add 永远不用 `-A`，逐个文件点名。

**3. 分支巡检定时任务。** 用 OpenClaw 的 cron 每周跑一次：列出已合并分支、超过两周未动的分支、工作区不干净的检出目录，汇总成报告推给我。删除动作仍由我确认后单独执行。

## 踩坑点

- **message 与 diff 不符。** diff 很大时 agent 会"脑补"提交意图，写出来的 message 看着漂亮但不准。解法是限制上下文：超过一定行数就先按文件分块总结，再合成总 message。
- **`git add -A` 是事故之源。** 有一次它把本地 `.env` 和构建产物一起暂存了，幸好流程要求确认文件清单才拦下来。`.gitignore` 要先补齐，skill 里也要显式禁止 `-A` 和 `.`。
- **交互式命令会挂死。** `rebase -i`、需要输密码的 remote 操作不要放行，agent 会一直等输入直到超时。
- **脏仓库状态。** 处于 rebase 中间态或 detached HEAD 时，agent 的判断会乱。流程第一步强制检查 `git status`，状态异常直接中止并报告。

## 可复用建议

- 所有写操作前必须有"提案—确认—执行"三步，确认信息里带完整命令和文件清单；
- 给 agent 的 git 访问限定在工作目录内，不要开全局 shell；
- 保留一份 agent 执行过的 git 操作日志，出问题能回溯；
- 白名单和流程模板写进 skill 文件并纳入版本管理，别靠聊天里的口头约定；
- 如果团队走 MCP 路线，思路相同：把 git 封装成 MCP server 里的受限工具，白名单放在 server 层强制执行，比靠 prompt 约束可靠得多。

## 总结

Git 自动化的收益不在"全自动"，而在把写 message、巡检分支这类低价值操作外包出去，同时用白名单和确认闸门守住边界。一个多月下来，我的提交历史干净了不少，死分支清零，没有发生过一次误提交。让 agent 做提案者，人做审批者——这个分工目前看是最稳的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/9e68859dc946e1a6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/7c3fd7c64f1c51a5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/bb04fc166a2dc8ea.png)

