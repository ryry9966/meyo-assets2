---
title: 让 AI 助手接管 Git 日常：提交信息、分支清理与安全护栏
feedId: 37585
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

日常开发里 Git 操作占比不小，但价值密度很低：写 commit message、开分支、合并后清理、准备 PR 描述。这些事规则明确、上下文局部，正好适合交给 agent。OpenClaw 的插件 / MCP 机制可以把 git 封装成一组细粒度工具，让助手在受控范围内打理仓库。

## 问题

直接给 agent 一个 shell 让它跑 git，效果差且危险：它会凭记忆编造 diff 内容、把 `.env` 一起 add、在共享分支上 rebase。问题不在模型能力，而在没有边界——工具越万能，事故越随机。

## 做法

1. **工具最小集**：只暴露 `status` / `diff` / `log` / `branch` / `add` / `commit` / `checkout -b` 等只读加低风险写操作，不暴露 `push --force`、`reset --hard`、`clean`。
2. **约定进指令**：Conventional Commits 格式、分支命名 `feat/xxx`、`fix/yyy`、单分支单任务，写成一段固定的 housekeeping 规则。
3. **流程固定**：agent 先跑 `status + diff`，基于真实输出草拟 commit message，列出待暂存文件清单，等确认后再 commit。
4. **分支生命周期**：任务开始建分支；合并后用 `git branch --merged` 找出可删分支，列清单人工勾选后删除。
5. **留痕**：所有 agent 的 git 写操作追加到日志文件，出问题能回溯是谁在什么时候动了什么。

## 踩坑点

- **凭记忆写 commit**：不要让 agent 复述"我刚才改了什么"，强制以 `git diff` 输出为唯一事实来源，否则会出现与代码无关的漂亮废话。
- **暂存失控**：`add -A` 是事故源头。改为逐文件列清单确认，密钥文件靠 `.gitignore` + 暂存前检查双保险。
- **改写已推送历史**：规则里明令禁止对已推送分支做 amend/rebase，整理只允许发生在本地未推送分支上。
- **中间态残留**：agent 会话中断可能留下 detached HEAD 或冲突现场。每次任务开头先跑 `git status`，有异常先报告再动手。

## 可复用建议

- 把整套约定做成一个 git-housekeeping 技能 / 提示词模板，换仓库零成本复用。
- 先 dry-run 一到两周：agent 只输出建议的 commit message 和分支操作清单，不执行。观察准确率后再逐步放开写权限。
- 确认门是底线：commit、删分支必须人工点确认，push 一律不交给 agent。自动化程度可以慢慢加，权限不能一次给满。

## 总结

AI 管 Git 的收益不是省打字，而是让提交历史和分支状态变得干净、可预期。关键在三点：细粒度工具、明确约定、确认门，而不是一个万能 shell。OpenClaw 的插件机制很适合做这种"窄权限"封装——从只读分析起步，用一段时间建立信任，再放权，是最省心的路径。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/cd8b43b1363d4f0f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/eb042f405f655b91.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/cf485f9ba25c0158.png)

