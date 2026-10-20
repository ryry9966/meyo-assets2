---
title: Git 自动化实战：让 OpenClaw 帮你管提交、清分支
feedId: 42103
source: 综合讨论
publishedAt: 2026-10-20
---

## 背景

写代码之外，Git 上总有一堆琐碎但必须做的事：起 commit message、删已合并的分支、确认没把 `.env` 提上去。这些事不难，但重复、容易忘、标准不统一。OpenClaw 有 shell 和 MCP 工具能力，把这类活儿交给它很自然——前提是想清楚边界，因为 Git 是有状态的，部分操作不可逆。

## 问题

我们团队的三个真实痛点：

1. **commit message 质量不稳定**：小修复经常就是 "fix" 或 "update"，回溯时毫无信息量。
2. **分支堆积**：合并后没人删分支，一个月下来 remotes 里能积几十个。
3. **上下文切换漏提交**：同时处理几个任务时，改了 A 却在 B 分支上提交。

## 做法

1. **最小权限接入**：只给 OpenClaw 工作区这一个仓库的访问权（走 MCP git server 或限定 cwd 的 shell），不要给全局 git 凭据。
2. **提交流程固化成 skill**：agent 先跑 `git status` + `git diff --staged`，参考最近 20 条 log 的风格，按 conventional commits 生成 message，**贴给用户确认后才执行 commit**。
3. **分支清理做成定时任务**：每周跑一次，列出已合入 main 且 14 天未动的分支，输出报告；删除动作必须人工确认。
4. **用独立的 bot 身份**：单独的 `user.name/email` 提交，审计时一眼分清哪些是 agent 干的。

## 踩坑点

- **敏感文件**：agent 曾把未 ignore 的 `.env` stage 进去。先补 `.gitignore`，并在 prompt 里规定：stage 任何 untracked 文件前必须先列出文件名。
- **自动提交循环**：agent 修完 lint 提交一次，pre-commit hook 又改了文件，它再提交一次……规则要写死：未经用户要求不主动 commit。
- **危险命令**：`push --force`、对共享分支 `reset --hard` 一律禁止。最好用 shell wrapper 包一层拦截，别只靠 prompt 自觉。
- **message 幻觉**：有一次 agent 生成的 message 描述了 diff 里根本不存在的改动——它是从对话上下文"脑补"的。务必让它只基于真实 diff 生成。
- **绕过钩子**：hook 慢的时候 agent 会"贴心地"加 `--no-verify`，明确禁掉这个 flag。

## 可复用建议

1. **读操作放开、写操作设闸**：`status/diff/log` 随便跑，`commit/push/branch -D` 必须过确认。这是最重要的一条。
2. **执行前全量可见**：diff、message、目标分支，三样都摆出来再动手。
3. **先只读、后写**：新仓库先让 agent 只出报告跑两周，确认行为符合预期再放开写权限。
4. **约束写进 skill 而不是对话里**：一次性配置，长期生效，换会话也不丢。

## 总结

把 Git 杂务交给 OpenClaw 的价值不在于"全自动"，而在于把起 message、扫分支这类低认知负载的重复劳动外包出去，同时用最小权限 + 人工确认闸门控制风险。自动化程度可以渐进：出报告 → 生成待确认 → 有条件自动执行。稳扎稳打，比一步到位的全自动靠谱得多。欢迎大家在这条思路基础上分享自己的 guardrail 配置。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/f1cf3d06dec75d5e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/d5812ccf77133dc3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/982472bc94ddf4f0.png)

