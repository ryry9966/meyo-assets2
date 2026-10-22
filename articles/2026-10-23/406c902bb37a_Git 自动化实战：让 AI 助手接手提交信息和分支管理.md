---
title: Git 自动化实战：让 AI 助手接手提交信息和分支管理
feedId: 42319
source: 综合讨论
publishedAt: 2026-10-23
---

## 背景

最近几个月，我的日常开发模式变成了“AI 写码、人做评审”。随之而来的是一个琐碎但高频的问题：一次会话里 agent 会改十几个文件，`git status` 一片红。手写 commit message 成了最机械的环节，分支也越堆越乱——`feature/xxx`、`fix-yyy`、`wip-zzz` 混在一起。

于是我花了一个周末，把提交和分支管理交给 agent，核心思路是：**低风险操作全自动，高风险操作设人工确认门**。跑了一个多月，稳定可用，把做法沉淀下来。

## 问题

直接把 git 权限丢给 agent，会遇到三类麻烦：

1. **提交质量不可控**：agent 倾向 `git add -A`，把调试代码、临时文件一起提交；message 还会“脑补”没做过的改动。
2. **分支状态混乱**：可能在 main 上直接 commit，或者把已推送的提交 amend 掉。
3. **上下文浪费**：大 diff 直接塞进上下文，token 烧得快，message 反而写得笼统。

## 做法

**第一步：收窄工具面。** 我没有给 agent 完整 shell，而是用 MCP git server（或白名单脚本），只暴露 `status`、`diff`、`add`、`commit`、`branch`、`log` 这几个只读或低危操作。push、reset、rebase 一律不开放，需要时自己在终端执行。

**第二步：写规则文件。** 在项目根目录给 agent 一份约定，关键三条：

- commit 前必须先跑 `git status` 和 `git diff --stat`，按逻辑分组提交，禁止一次性 add 全部
- message 用 Conventional Commits，正文写“为什么”，不写“改了哪些文件”
- 分支命名 `agent/日期-主题`，禁止在 main 上直接提交

**第三步：设置确认门。** 工作流里加一个硬性节点：commit 可以自动执行，但执行后必须把 hash 和 message 贴出来等我确认；涉及分支切换时先报当前分支。配合 pre-commit hook 跑 lint，格式问题在钩子层兜底。

**第四步：处理大 diff。** 文件多时让 agent 先 `diff --stat` 总览，再按模块分片读 diff，每组改动单独 commit。message 更准，回滚粒度也更细。

## 踩坑点

- **`git add -A` 是万恶之源**。规则里必须显式禁止，否则 `.env`、日志文件很容易混进去。我在 `.gitignore` 之外又加了 pre-commit 的敏感文件检查。
- **message 幻觉**。agent 有时会描述 diff 里根本不存在的重构。要求它逐条引用改动文件后再写 message，幻觉率明显下降。
- **amend 已推送提交**。有次 agent 为了“让历史更整洁”去 amend，幸好 push 不在它的权限里。这条要写进规则：已推送的提交只允许追加 fixup。
- **上下文超限**。两千行的 diff 直接读会爆，务必先 `--stat` 分片。

## 可复用建议

- 权限分层表可以直接抄：只读命令自动执行；本地写操作自动执行＋事后确认；网络操作（push）永远人工。
- 规则文件控制在 20 行以内，太长 agent 反而抓不住重点。
- 让 agent 每次会话结束输出一份“分支状态摘要”：未提交改动、领先/落后提交数，人只做决策。

## 总结

AI 助手做 Git 自动化，价值不在“全自动提交”，而在于把提交信息撰写、分支整理这类机械劳动接管掉，同时用权限分层守住安全边界。一个多月下来，我的 commit 历史第一次变得工整，回滚也从考古变成了看日志。建议从小范围起步：先只开放 message 生成，跑顺了再逐步放开分支操作。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/bcfeb02927fbce15.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/d651027725145572.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/3347433e35c29d0b.png)

