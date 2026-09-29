---
title: 把 Git 杂活交给 Agent：提交与分支自动化的落地笔记
feedId: 39513
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

日常开发里，Git 操作大半是机械劳动：想 commit message、按规范建分支、清理已合并分支、push 前检查状态。这些活规则明确、上下文全在本地仓库，恰好是 Agent + MCP 工具擅长的场景。最近我把这部分逐步交给跑在 OpenClaw 里的助手，稳定运行几周后，记录一下做法和坑。

## 问题

- commit message 全是 `fix`、`update`，半年后没人知道当时改了什么；
- 分支越堆越多，已合并的没人删；
- 下班前常忘提交，第二天上下文全丢；
- 但直接让 Agent 执行任意 git 命令又太危险：`reset --hard`、`push --force` 跑错一次就是事故。

## 做法

1. **接工具、收权限**。用 MCP 的 git server（或限定在仓库目录的 shell），只开放 `add / commit / branch / switch / log / diff / status / stash`，其余一律不进工具白名单。
2. **规则写进指令文件**。在 workspace 的 `AGENTS.md` / `TOOLS.md` 里固化：Conventional Commits 格式、`feat/xxx`、`fix/xxx` 分支命名、禁止命令清单（`reset --hard`、`clean -fd`、`push --force`、对共享分支 rebase）。
3. **定义提交流程**：`status` → `diff --stat` 看规模 → 分块看关键文件 diff → 生成 message → 提交。本地 commit 已放开，push 不放。大 diff 只看 stat，防止上下文爆炸。
4. **加定时任务**。每天下班前自动汇总当天改动、列出未提交文件清单，问我要不要提交；每周列一次已合并分支，确认后清理。

## 踩坑点

- **上下文截断导致幻觉**：整仓库 diff 直接塞给模型，它会“补写”没改过的内容。先 `--stat` 概览再分块看，message 质量稳定很多。
- **脏工作区切分支**：Agent 在有未提交改动时执行 `switch`/`rebase`，现场直接搞乱。后来加了一条硬性前置检查：工作区不干净，禁止动分支。
- **凭据安全**：token 别写进提示词或明文配置，走系统 credential helper / ssh-agent，仓库路径做白名单。
- **归属不清**：Agent 生成的 commit 一律加 `Co-authored-by` 标记，review 时能分清哪些是机器写的。

## 可复用建议

- **先只读，再写入**。第一周只让它看和报告，确认 message 质量稳定后再放开 commit 权限，逐步灰度。
- **本地自动化、远程手动**。push 永远要人确认，这是性价比最高的安全边界。
- **规范是自动化的前提**。团队本身没有 commit 约定，Agent 写出来的也只是另一种随意的 message。

## 总结

让 Agent 管 Git，价值不在炫技，而在消掉“想起来要提交”这类摩擦。规则前置、权限收窄、危险操作留人工——这套下来每天能省十几分钟，更重要的是，仓库历史终于可读了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/f7e6cf9549cf9c53.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/03b0d9848b3ce79f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/86d047f3906858c9.png)

