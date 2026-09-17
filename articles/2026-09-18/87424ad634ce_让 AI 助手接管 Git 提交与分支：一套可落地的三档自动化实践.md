---
title: 让 AI 助手接管 Git 提交与分支：一套可落地的三档自动化实践
feedId: 38017
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

日常开发里相当一部分 Git 操作是低认知、高重复的：写完功能补 commit message、删掉已合并分支、定期清理远端遗留分支。这些事不难，但频繁打断心流。OpenClaw 这类 agent + MCP 的组合恰好适合接住这类活：Git 操作有结构化的输入（diff、分支列表）和可校验的输出（提交是否成功、分支是否删除），自动化收益明确。

## 问题

直接“让 AI 帮我 commit”会遇到几个具体麻烦：

1. 提交信息质量不稳定，容易编造 diff 里不存在的内容；
2. `git add -A` 一把梭，把无关文件带进提交；
3. 分支清理涉及远端和团队协作，误删成本高；
4. agent 的 Git 权限如果过大，一次错误指令就可能 force push。

核心矛盾是：自动化程度越高，出错半径越大。

## 做法

我把风险拆成三档，逐档放开：

**第一档（零风险）：commit message 生成。** agent 只读 `git diff --staged`，输出符合 Conventional Commits 的信息，人确认后提交。走 MCP git 工具或 shell 都行，关键是 prompt 里明确“只允许描述 diff 中出现的变化，禁止推测意图”。

**第二档（低风险）：本地分支清理。** 定时任务每天跑一次：列出已合并进 main 的本地分支，agent 汇总成“可删除清单”，先 dry-run 输出，人工确认后执行 `git branch -d`。注意用 `-d`（已合并才删），不要用 `-D`。

**第三档（需审计）：提交后自动动作。** 比如自动推送、按路径打标签。这一档开始必须记录 agent 执行的每条命令，最好通过只暴露白名单命令的 MCP git server 来做，而不是给 agent 一个裸 shell。

系统提示词大致结构：

```
角色：Git 助手
输入：staged diff / 分支列表
规则：
- 只引用 diff 中存在的内容
- 禁止 force push，禁止操作 main
- 执行前列出将要运行的命令
```

## 踩坑点

- **diff 过大导致幻觉。** Monorepo 一次提交上千行，上下文装不下 agent 就开始编。解法：按文件分块摘要，或只喂 `--stat` 加关键文件 diff。
- **白名单比提示词可靠。** 光在 prompt 里写“不要 force push”不够，工具层直接不提供该能力才是硬约束。
- **凭证最小化。** 给 agent 的 SSH key / token 只开单仓库读写，别复用个人主账号。
- **合并策略先定。** 让 agent 自动 rebase 之前，先确认团队策略，否则历史会被搅乱。

## 可复用建议

- 推进顺序：commit message → 本地分支清理 → 远端操作，每档稳定跑两周再放开下一档。
- 一切自动化先 dry-run，“将执行的命令”和“执行结果”分开输出，人工确认作为默认环节。
- 工具面收窄：一个只含 status / diff / commit / branch / list / delete 的工具集，好过完整 Git 权限。
- 留审计日志：每条 Git 命令落盘，出问题能回放。

## 总结

Git 自动化本身不难，难的是给 agent 划清权限边界。经验是：让 AI 负责“读和写提案”，让人负责“确认和执行高危动作”。这样既省掉重复劳动，又把出错半径控制在可回滚范围内。这套三档推进的思路，同样适用于其他需要操作生产资源的自动化场景。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/933c2f4eb77010ac.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/78d17d8d03c9f414.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/aa381bfb245f32fa.png)

