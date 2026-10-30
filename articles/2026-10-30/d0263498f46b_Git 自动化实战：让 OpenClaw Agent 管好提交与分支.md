---
title: Git 自动化实战：让 OpenClaw Agent 管好提交与分支
feedId: 43049
source: 综合讨论
publishedAt: 2026-10-30
---

## 背景

日常开发里有一类活儿没什么技术含量但很占时间：写提交信息、切分支、清理合并后的旧分支、攒 changelog。这些操作模式固定、上下文就在仓库里，天然适合交给 AI Agent。我在 OpenClaw 里接了一个带 shell 和 Git MCP 工具的 Agent，跑了一个多月，把收敛下来的做法和踩过的坑记录一下。

## 问题

直接把 git 权限丢给 Agent 有三个真实风险：

1. **破坏性操作**：`reset --hard`、`push --force`、`clean -fd` 在错误时机被执行，损失的是真实代码；
2. **幻觉提交**：diff 被截断或上下文不足时，Agent 会编造提交信息，把"修复配置"写成"重构认证模块"，污染历史；
3. **误伤工作区**：`git commit -a` 会把无关的本地改动一起卷进提交。

## 做法

### 1. 沙箱克隆，不动主工作区

给 Agent 单独 `git clone` 一份仓库作为工作区，主工作区永远归人。Agent 的所有操作先落在克隆里，人确认后再 push。爆炸半径立刻变小。

### 2. 命令白名单 + 两段式执行

在系统提示里明确边界：允许 `status / diff / log / add / commit / branch / switch / merge --ff-only`；禁止 `push --force / reset --hard / clean / rebase -i`。所有写操作先输出待执行的命令清单，人确认后再执行。看着繁琐，但救过我好几次。

### 3. 提交信息生成

提示词模板约定 Conventional Commits 格式，外加两条硬规则：diff 超过 400 行时先用 `--stat` 概览、再按文件定点看；语义拿不准时降级为 `chore:`，不许猜。stage 只用显式文件列表，禁用 `-A`。

### 4. 分支清理与 changelog 走"报告制"

定时任务每周让 Agent 跑 `git branch --merged main` 加最后提交时间，产出待清理分支报告；changelog 聚合上个 tag 以来的提交、按 type 分组。两者都只生成报告，删除分支和打 tag 由人执行。

## 踩坑点

- **hook 失败后的盲目重试**：husky/lint 挂掉时 Agent 会反复重试，把仓库越搞越乱。规则必须写死：hook 失败即停，把原始错误带回来给人。
- **dirty worktree**：切分支前先跑 `git status --porcelain`，非空就中止，否则未提交改动会被带到错误分支上。
- **上下文膨胀**：全量 diff 直接进上下文，输出会变得不可信。始终先 `--stat` 再定点读。
- **交互式命令卡死会话**：任何等待 stdin 的 git 命令都会挂起，模板里统一用非交互参数（`-m`、`--no-edit`）。

## 可复用建议

- 提示词模板放进仓库（如 `.agent/git.md`），版本化管理，团队共用；
- 所有 Agent 的 git 操作落一份日志，事后可审计；
- "报告先行、确认后执行"的两段式模式不限于 git，一切有副作用的自动化都适用；
- 分支保护是最后一道防线，Agent 的 push 权限只覆盖非保护分支。

## 总结

Git 自动化的价值不在"全自动"，而在把机械部分交给 Agent、把决策权留给人。沙箱克隆、命令白名单、两段式确认这三件事落地后，这套流程稳定跑了六周，没有出现过一次需要手工救回的提交。建议从提交信息生成这个最低风险场景起步，再逐步开放分支管理，最后才是 tag 和合并——权限的开放速度，永远慢于你对 Agent 的信任速度。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/13267171723733f0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/874d13c2587ac712.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/b18269bc054f204a.png)

