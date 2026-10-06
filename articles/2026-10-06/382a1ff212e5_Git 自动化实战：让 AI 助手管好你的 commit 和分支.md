---
title: Git 自动化实战：让 AI 助手管好你的 commit 和分支
feedId: 40675
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

在 OpenClaw 这类 agent 工作流里，让 AI 助手直接操作 Git 是收益最明显的一步：commit message 生成、分支整理、cherry-pick、冲突预处理，都是重复度高、规则明确、判断成本低的活。但真正把 agent 接进 Git 后你会发现，"能用"和"敢用"之间差着一层安全设计。这篇记录我在自己的仓库上跑通的一套做法，供参考。

## 问题

手工维护 Git 的三个老痛点：

1. commit message 质量不稳定，连续小提交时尤其敷衍；
2. feature 分支只增不减，本地堆几十个，不知道哪些其实已经合并；
3. agent 有了 shell 权限之后，最怕它手滑执行 `reset --hard` 或 `push --force`，这类操作不可逆。

## 做法与步骤

**第一步：收敛权限。** 不要给 agent 全量 shell。我用 MCP 的 git server，只暴露 `status` / `diff` / `add` / `commit` / `branch` / `log` 这几个工具，push 和 reset 一律不开放，需要时人工执行。

**第二步：写死提交流程。** 在 agent 的 skill 文件里固化流程：先 `git status` + `git diff --stat` 看概览，再按需读局部 diff，最后按 Conventional Commits 生成 message。关键是要在提示词里给足规范示例，否则它会输出 "update files" 这种没有信息量的废话。

**第三步：分支自动整理。** 每周让 agent 跑一次任务：列出本地分支、对比 origin 判断哪些已合并、输出一张清理建议清单。注意这一步只做"报告 + 建议"，批量删除动作放人工审批。

**第四步：dry-run 习惯。** 任何涉及历史的操作（rebase、cherry-pick），先让 agent 给出执行计划和影响范围——涉及哪些 commit、可能冲突的文件，确认后再放行。

## 踩坑点

- **大 diff 撑爆上下文**：一次改 50 个文件时直接喂全量 diff 会溢出，先 `--stat` 再挑关键文件看局部。
- **agent 混淆文件状态**：untracked 文件有时会被它当成"已修改"处理，流程里强制它区分 staged / unstaged / untracked 三类。
- **detached HEAD 误提交**：rebase 中途让 agent 提交，commit 会丢在游离 HEAD 上。规则写死：agent 发现当前不在任何分支上，必须停下报告。
- **中文 commit 与 commitlint 冲突**：仓库启用了 commitlint 时，让 message 的 type/scope 保持英文、description 可中文，提前在提示词里写明，避免反复打回。

## 可复用建议

- 把"Git 操作守则"做成独立 skill 文件挂载到所有 agent 会话，比散落在各处 system prompt 里好维护；
- 白名单工具集 + 人工审批高危操作，是 agent 接 Git 的底线配置，别省；
- 要求 agent "计划先行"：先说打算执行什么命令、为什么，再动手。这一条在 Git 场景外的自动化里同样适用。

## 总结

AI 管 Git 的价值不在"全自动"，而在把判断和执行的分工划清楚：agent 负责收集信息、生成规范 message、给出整理建议；人负责所有不可逆操作。权限收窄、流程固化、dry-run，这三件事做齐，agent 操作 Git 才能从提心吊胆变成日常可用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/7ca4f995fe4eb4e0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/1a6c4e41f0008e05.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/dcbabcbfdcbfe92f.png)

