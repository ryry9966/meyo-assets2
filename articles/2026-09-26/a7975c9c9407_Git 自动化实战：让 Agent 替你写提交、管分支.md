---
title: Git 自动化实战：让 Agent 替你写提交、管分支
feedId: 39105
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

日常开发里，Git 操作占了不少碎片时间：写 commit message、切分支、清理已合并分支、暂存改动。这些事不难，但琐碎、频繁、容易被打断。OpenClaw 这类常驻 Agent 的出现让"把杂活交给助手"变得现实——它有 shell 执行能力，能挂 MCP 工具，还能长期记住你的规范。这篇帖子记录我在个人仓库上跑了一个多月的做法，供参考。

## 问题

手工模式下有几个真实痛点：

1. commit message 质量不稳定，赶进度时全是 `fix`、`update`，回溯时没有信息量；
2. 分支堆积，feature 分支合完忘删，两周后 `git branch` 一屏放不下；
3. 上下文切换成本高，改到一半被叫去修别的问题，stash 一次就容易丢东西。

## 做法

我的方案分三层，都不复杂：

**第一层：定权限边界。** 只对 Agent 开放 git 只读命令（`status` / `diff` / `log` / `branch`）和一个受限的提交入口。push 和 rebase 默认关闭，需要我在对话里明确说"推上去"才执行。`main`、`release` 等保护分支写进 skill 的禁改清单。

**第二层：写一份提交规范 skill。** 核心规则：提交前必须先读 `git diff --staged` 的真实改动；message 用约定式格式（`type(scope): summary`），summary 不超过 50 字符，正文说明动机而不是罗列文件名；禁止在没看 diff 的情况下编 message——这条专门治幻觉。

**第三层：分支生命周期自动化。** 让 Agent 每天跑一次巡检：列出已合并到 main 的本地分支，生成"建议删除"清单，我确认后它再执行 `git branch -d`（小写 d，强删走不到）。新分支命名也交给它，按 `feat/日期-关键词` 规则生成，避免 `test2`、`final-final` 这种名字。

## 踩坑点

- **diff 太大时 Agent 会偷懒概括。** 单次改动几十个文件时，它给出的 message 会失真。后来要求按文件分块读 diff，超过阈值就建议拆成多次提交。
- **本地钩子会挡住 Agent 的提交。** pre-commit 跑 lint 失败时，Agent 可能自作主张加 `--no-verify`。skill 里明令禁止，失败就停下来报告。
- **detached HEAD。** 在 CI 检出目录或 worktree 里操作时容易踩，commit 会悬空丢失。现在流程第一步固定检查当前分支状态。
- **commit 签名。** 机器没配 GPG/SSH 签名环境时，代提交的签名会缺，过不了远端策略。要么提前配好，要么在 skill 里声明降级并知会团队。

## 可复用建议

1. 先只读后写入：让 Agent 跑一周"只建议不执行"，观察判断质量，再逐步放开 commit 权限；
2. "提交前必读 diff"值得写死在 skill 里，比任何 prompt 技巧都管用；
3. 破坏性操作（`push -f`、`branch -D`、`reset --hard`）一律放白名单外，人工触发；
4. 分支巡检做成定时任务，输出到固定文件或频道，形成可审计记录。

## 总结

这套东西没有用到高深技术，本质是三件事：**权限收敛、规范固化、人工确认点**。跑了一个多月，commit message 可读性明显好转，本地分支稳定在个位数。Agent 替代不了的是判断——哪些改动值得拆、哪些提交该 revert——这些仍然留在人手里，也应该留在人手里。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/6ba0c87fc7b3bc74.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/bee97d8d6774d7b9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/b04132b92c49935b.png)

