---
title: Git 自动化实战：让 AI 助手管提交和分支，但别给它踩雷的机会
feedId: 37551
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

Agent 能写代码、跑测试之后，Git 这层反而成了短板：提交信息随手写、feature 分支只增不减、合并完忘了清理。我们在 OpenClaw 工作流里把 Git 操作收敛给 Agent，跑了几个月，把做法和教训整理如下。

## 问题

三个实际痛点：

1. **提交信息质量不可控**。人和 AI 产出的 commit 混在一起，回溯时基本靠猜。
2. **分支生命周期无人管**。远端堆了上百个已合并分支，谁都不敢动手删。
3. **权限风险**。Agent 一旦持有完整 git 权限，`push --force`、`reset --hard` 这类操作没有兜底，出事就是事故。

## 做法

方案是「MCP 工具收窄 + 提示词规范 + hook 兜底」三层结构：

**第一步：收窄工具面。** 接入 MCP Git server 时只暴露 `status`、`diff`、`add`、`commit`、`branch`、`log` 等低风险工具；`push` 走独立确认通道，`push --force`、`reset --hard` 直接不注册。

**第二步：固化提交规范。** 系统提示词里写死 Conventional Commits 格式，scope 只允许几个固定值，正文要求写动机而不是罗列改动。规范文件放仓库里，人和 Agent 共用一份。

**第三步：提交流程拆两段。** Agent 先基于 staged diff 输出「变更摘要 + 建议提交信息」，人工确认后才执行 commit。多一步确认看起来慢，实际拦截过好几次不相关文件混入。

**第四步：分支清理做成半自动任务。** 定时任务让 Agent 列出已合并到 main 的本地和远端分支，生成清理清单，确认后批量 `git branch -d` 和 `git push origin --delete`。

**第五步：hook 兜底。** `commit-msg` hook 用正则校验提交格式，服务端 `pre-receive` 拦截保护分支的 force push。这层与 Agent 无关，是最后防线。

## 踩坑点

- **敏感文件被误 stage**。Agent 有一次把未忽略的 `.env` 加进了暂存区。提示词约束不可靠，后来改成工具层路径黑名单，命中即拒绝。
- **大 diff 超上下文**。单次提交跨几十个文件时，让 Agent 先按模块聚合摘要，别整段塞，否则提交信息会漏关键变更。
- **rebase 冲突解得太"省事"**。Agent 倾向直接取 ours/theirs，现在冲突文件强制人工过一遍。
- **自动删分支误删过 pre-release 分支**。清理逻辑必须排除「已合并但未打 tag」的分支，这条规则是教训换来的。

## 可复用建议

1. 权限最小化 + 关键节点人工确认，比在提示词里写"请不要 force push"有效一个数量级。
2. Git 规范落成仓库内的 Markdown，Agent 读同一份，避免两套标准。
3. hook 是与 Agent 解耦的兜底层，半小时配置，长期受益。
4. 一切删除类操作：先生成清单、人确认、再执行，不要做成一键全自动。

## 总结

Git 自动化的核心不是让 Agent 替你敲命令，而是让它成为守规范的协作者：Agent 负责读 diff、写摘要、跑清理，你负责确认和兜底。工具收窄、规范共享、hook 兜底——这套三层结构在我们仓库稳定跑了三个月，commit 历史终于能看了。欢迎在评论区交流你们的权限配置方案。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/0c170b7d9b089876.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/b627e5ff560e73f9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/74d898eefd0d1f82.png)

