---
title: Git 自动化实践：提交信息交给 AI 助手写，push 留给自己按
feedId: 40424
source: 综合讨论
publishedAt: 2026-10-04
---

## 背景

OpenClaw 这类 agent 的核心能力是执行 shell 命令，Git 几乎是它最顺手的落点。我让助手接管了两件低价值、高重复的事：写提交信息、清理分支，跑了两个多月。效果不错，但前提是把边界划清楚。

## 问题

- 提交信息质量参差，赶时间就 `update`、`fix` 一把梭，事后翻 log 全靠猜；
- 分支越积越多，`git branch` 一屏放不下，哪些已合并根本记不清；
- 让 agent 裸跑 git 命令风险大：误提交 `.env`、误推 `--force`、`reset --hard` 丢工作区。

核心矛盾在于：agent 的效率来自自动执行，但 Git 的写操作恰恰是“出事没有撤销键”的那一类。

## 做法

**第一步：只读操作全放开。**
让 agent 自由执行 `status` / `diff` / `log` / `branch -a`，并要求它每次动手前先汇报“当前在哪个分支、暂存区有什么”。这一步几乎零风险，先把信息收集自动化。

**第二步：提交信息走“暂存区 diff + 规范模板”。**
prompt 里固定两条：只看 staged diff（不是工作区全部改动），输出遵循 conventional commits。经验是 staged diff 生成的 message 准确率高很多——agent 不会把还没决定要不要提交的改动也写进去。

**第三步：写操作加白名单。**
在我自定义的 git skill 里明确分级：

- 允许：`add`、`commit`、创建分支；
- 需确认：`merge`、`rebase`、push 到非个人分支；
- 禁止：`push --force`、`reset --hard`、`clean -fd`。

执行前 agent 必须贴出完整命令和涉及的文件列表。

**第四步：分支清理做成例行任务。**
每周让 agent 跑一遍 `git branch --merged`，列出可删分支，生成清单等人确认，命名不合规的一并标记。

## 踩坑点

1. agent 会“顺手”把未跟踪文件全部 `add`，包括 `.env` 和临时文件。规则里必须要求它先列出待暂存文件、逐个确认。
2. 生成的信息偶尔会“脑补”改动意图——diff 只有改了什么，没有为什么。涉及业务语义的提交，我会额外补一句变更动机，效果立竿见影。
3. `pull` 产生的 merge commit 很吵。让它默认 `pull --rebase`，但仅在确认本地无未推送提交时执行。
4. 长会话里 agent 会忘掉之前的分支状态。关键操作后让它重新跑一次 `status` 刷新认知，别依赖记忆。

## 可复用建议

- 原则一句话：**读操作自动化，写操作人门禁，破坏性操作直接禁。**
- 把 Git 规则沉淀成 skill/插件，而不是每次贴 prompt。规则写一次，所有会话生效。
- 要求 agent 输出它执行过的每条命令，事后可审计。
- 渐进开放：先只做提交信息，稳定两周再放分支管理。

## 总结

Agent 管 Git 的价值不在“全自动”，而在消掉机械劳动、把判断留给人。提交信息让它写，push 让自己按——这个分配比例是我试出来的最舒服的姿势。规则沉淀成 skill 之后，同事的 agent 拿来就能用，这大概是最实在的收益。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/0b932f6499f66f3b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/b38a662ebe183d19.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/b13c69593410abab.png)

