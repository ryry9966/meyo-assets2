---
title: 让 AI 帮你管 Git：提交与分支自动化的落地实践
feedId: 37435
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

用 OpenClaw 这类 Agent 写代码后，很快会遇到一个尴尬的分工：AI 负责改代码，人负责敲 git。一天下来工作区堆了十几个文件的改动，提交信息全是"update"，分支开了忘删。既然 Agent 已经能读写代码库，把 Git 的机械操作也交给它是自然的一步——前提是管得住。

## 问题

把 git 权限直接丢给 Agent，通常会出三类事：

1. **一锅端提交**：`git add -A` 加一句"update files"，重构、修 bug、改配置混在一个 commit 里，事后回溯基本作废。
2. **幻觉式提交信息**：Agent 没认真读 diff，message 描述的是它"以为"自己做的事。
3. **危险操作**：对已推送的分支 amend/rebase、直接 push 到 main、force push。

## 做法

我的落地路径分四步：

**1. 受控接入。** 通过 MCP git server 或沙箱 shell 提供 git 能力，只暴露需要的工具：status、diff、add、commit、branch、log。push、`reset --hard` 这类默认不给。

**2. 规则写进仓库。** 在项目的 AGENTS.md（或等效配置）里明确：提交信息用 Conventional Commits；分支命名 `feat/<task-id>-摘要`；禁止改写已推送历史；main 只读。规则跟着代码走版本控制，比散在对话里可靠得多。

**3. 固化工作流。** 让 Agent 每次提交前走固定流程：`git status` → 逐文件读 diff → 按逻辑分组 stage（重构归重构，修复归修复）→ 基于真实 diff 写 message → 提交前跑 lint/test → 输出变更摘要等人工确认。

**4. 权限分级。** commit 允许自动执行；push、merge、rebase 停下来问人。跑稳定后再逐步放开，比如允许 push 到自己的任务分支。

## 踩坑点

- **stage 粒度**：不约束的话 Agent 还是爱用 `git add -A`。规则里明确"按文件甚至按 hunk stage，一次提交只做一件事"后明显改善。
- **遗留状态**：Agent 偶尔把工作区留在 merge 冲突或 detached HEAD 就去干别的了。工作流末尾加一步"确认 `git status` 干净"，或者每个任务用独立 worktree 隔离。
- **凭据暴露**：能碰 shell 的 Agent 理论上能翻到 `.git/config` 或 credential helper 里的 token。用系统级 credential helper，别把 token 写进 remote URL，并收敛 Agent 可访问的目录范围。
- **审计缺失**：Agent 执行过哪些 git 命令，事后说不清。在工具层包一层日志，每条命令落盘，出问题能回放。

## 可复用建议

- 规则文件放仓库里，团队共享、随代码演进，不塞在一次性对话里；
- 一个任务一个分支一个 worktree，多 Agent 并行不互相踩；
- 先只读跑一两周，观察它的 diff 阅读和分组判断，再放写权限；
- 把"提交前必须通过 lint/test"做成 pre-commit hook，别依赖 Agent 自觉。

## 总结

让 AI 管 Git 的价值不在于"全自动"，而在于它能把提交拆干净、把信息写规范，把人从机械操作里解放出来去做真正的 review。权限分级加审计兜底，这套流程在我们仓库跑了一个多月，提交历史可读性提升明显，事故零起。工具交给 Agent，合并键留在人手里——这个边界目前看是对的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/4efc289a2f12692d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/e7dcd3be46e5f9da.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/e12751347232d622.png)

