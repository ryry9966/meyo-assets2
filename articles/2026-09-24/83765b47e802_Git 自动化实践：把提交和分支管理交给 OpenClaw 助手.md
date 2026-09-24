---
title: Git 自动化实践：把提交和分支管理交给 OpenClaw 助手
feedId: 38828
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

用 Agent 写代码的日常大概是这样：对话里描述需求，助手改文件、跑测试，最后留下一堆改动等你收拾。写 commit message、切功能分支、清理陈旧分支，这些琐事占掉的时间不比写代码少。OpenClaw 这类常驻 agent 接上 MCP 的 git/filesystem 工具之后，这些活其实可以交给助手做——前提是规矩要立好。

## 问题

直接说一句"帮我提交"，通常会出三类事故：

1. **一把梭 add**：`git add -A` 把临时文件、日志甚至 `.env` 全带进去；
2. **message 与实际改动脱节**：模型按"猜"写描述，diff 里根本没有那回事；
3. **危险操作不设防**：rebase、`push --force`、在 detached HEAD 上乱提交，出了问题很难回滚。

根因在于：agent 拿到的是通用 shell 权限，而 git 的安全性完全依赖操作者清楚自己在哪个仓库、哪个分支、提交了什么。

## 做法

我分三步逐步放权：

**第一步：工具与约定。** 给 agent 配两个工具就够：受限 shell（或 MCP git server）和文件读写。同时在仓库里放一份 `AGENTS.md`，写死约定：分支命名 `feat/xxx`、commit 遵循 Conventional Commits、禁用 `-A`、push 前必须过测试。会话启动时助手会读到这份文件，相当于发了一本操作手册。

**第二步：只读生成，人工落笔。** 初期只让它做两件事：跑 `git status` + `git diff` 后生成 commit message 草稿，以及给出分支切分建议。我确认后再执行。这个阶段主要是校准它的判断力。

**第三步：半自动执行。** 校准后放开执行权，但加硬护栏：

- 提交前助手必须先输出"计划"：当前分支、待提交文件清单、message——我回复确认才动手；
- 只 add 计划里列出的文件，杜绝 `-A`；
- pre-commit 钩子里跑密钥扫描，兜底防泄密；
- `push --force`、`reset --hard`、改写已推送历史，直接进黑名单。

跑顺之后，日常流程变成：改完代码说声"提交"，助手给计划，确认后自动 commit；每周让它跑一次分支清理，列出已合并的本地分支和过期远程分支，确认后删除。

## 踩坑点

- **不校验 detached HEAD**：有一次它 review PR 时被 checkout 到 detached 状态，接着直接 commit，提交"悬空"了。之后约定：任何写操作前必须先执行 `git branch --show-current` 并回报。
- **message 过度美化**："重构了部分逻辑"会被写成"全面重构核心模块"。要求它引用 diff 中的关键函数名来组织描述后，可信度明显上升。
- **交互式命令卡死**：`git rebase -i`、merge 冲突提示这类需要 stdin 的命令会让 agent 挂起。统一改用非交互参数（`--no-edit`、`merge --ff-only` 等）。
- **确认疲劳**：什么都要回 "ok"，很快我就开始无脑确认。所以确认点只留两个：push 和删除，其余自动执行。

## 可复用建议

1. git 约定写成仓库内文档，比全局提示词更可靠——它随代码走，团队共享；
2. 高频操作封装成 skill（`/commit`、`/branch-cleanup`），减少口头描述的歧义；
3. 写操作前强制"输出计划 → 确认 → 执行"三段式，dry-run 成本极低；
4. 护栏做在钩子和黑名单里，不要指望模型自觉。

## 总结

Git 自动化的价值不在"全自动"，而在把机械劳动交给助手、把判断权留给自己。先只读、再执行、护栏前置——这套节奏跑下来，省下的不只是每天几分钟提交时间，还有"刚才那坨改动到底提交了啥"的隐忧。建议从一个仓库开始试点，别一上来就接管所有项目。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/13ae53e7b01cf425.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/bc81d6e4101f4dcf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/f344601b713b1f0f.png)

