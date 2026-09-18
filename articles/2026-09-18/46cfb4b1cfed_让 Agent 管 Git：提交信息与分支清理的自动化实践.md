---
title: 让 Agent 管 Git：提交信息与分支清理的自动化实践
feedId: 38091
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

日常开发里最耗时间的往往不是写代码，而是那些琐碎但必须做的 Git 操作：起分支、写提交信息、合并后清理本地分支、补 PR 描述。这些事不难，但天天做，而且很容易做得不一致。OpenClaw 的 Agent 有 shell 访问能力和仓库上下文，天然适合接手这类任务——前提是把边界划清楚。

## 问题

我们团队遇到的实际痛点：

1. 提交信息质量参差，`fix`、`update` 满天飞，回溯问题时基本靠猜；
2. 本地堆了几十个 `feat/xxx`、`fix/xxx` 残留分支，谁也不敢乱删；
3. 重复操作打断心流，每次切换上下文的成本比操作本身还高。

关键矛盾在于：Git 操作出错的代价高（丢提交、强推覆盖），却又琐碎到不值得人脑处理。

## 做法

**第一步：给 Agent 配最小权限的 Git 工具。** 通过 MCP 的 git server（或受控 shell），但只放行本地命令：`status`、`diff`、`log`、`commit`、`branch -d`、`checkout`。`push`、`push --force`、`reset --hard`、`clean` 一律不进白名单，需要时人工执行。

**第二步：把规范写成文件，而不是写进提示词。** 在仓库根目录放一份 `GIT_STYLE.md`：分支命名（`feat/issue-123-short-desc`）、提交信息格式（Conventional Commits + 中文描述）、哪些文件不许进提交。Agent 每次先读这个文件再动手，规范改动走 code review，而不是改提示词。

**第三步：把三个高频动作封装成独立技能。**

- **提交信息生成**：只基于 `git diff --staged` 生成，禁止 Agent 凭记忆补充"为什么改"。生成后先打印出来确认，再执行 commit。
- **分支清理**：规则固定为"已合并进 main 且超过 14 天的本地分支"，用 `git branch --merged main` 列出，排除 `main` / `develop` / `release/*`，列清单等确认后再 `-d`（永远不用 `-D`）。
- **变更摘要**：开工前让 Agent 总结未提交改动和最近提交脉络，用于续接昨天的工作。

**第四步：所有 Agent 的 Git 动作留日志。** 一个 append-only 文件记录时间、命令、结果，出问题能回放。

## 踩坑点

- **提交信息幻觉**：早期让 Agent 看 diff 直接 commit，它会把 diff 里没有的"动机"写进去——明明是顺手重构，却写成"修复性能问题"。改成"信息必须逐条对应 diff 中的实际改动"，并在确认环节人工过目，才稳下来。
- **`--merged` 之外的分支不能碰**：有一次 Agent 判断"这个分支看起来完成了"，要删一个未合并分支，被我拦下。现在规则里明确写了：不在 `--merged` 输出里的分支，无论 Agent 多确定，一律不删。
- **大 diff 撑爆上下文**：几百个文件的提交直接读 diff 会爆。改成先 `git diff --stat` 看概览，再分块读关键文件。
- **Hook 里调 Agent 会死循环**：commit-msg hook 触发 Agent 校验信息，Agent 自己又执行了一次 commit。Hook 场景下必须禁用一切写操作。

## 可复用建议

1. **先只读，后写入**。让 Agent 跑两周只读操作（status / log / diff / 摘要），建立信任后再开放 commit 和 `branch -d`。
2. **规则进文件，提示词只管流程**。规范跟着仓库走，换人换机器都不跑偏。
3. **删除类操作必须"列清单 → 人确认 → 执行"三步走**，不给 Agent 一步到位的权限。
4. **远程操作（push、rebase 公共分支）不自动化**。这是底线，自动化收益覆盖不了事故成本。

## 总结

这套东西跑了一个多月，提交信息的一致性问题基本消失，分支清理从"想起来才做"变成每周自动提醒。核心经验就一句：让 Agent 处理琐碎和重复，把判断权和危险操作留在人手里。Git 自动化的目标不是"全自动"，而是"零意外"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/668dece34107bdd0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/360a678984d5b75a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/d9ad5803cdc182af.png)

