---
title: Git 自动化实践：让 OpenClaw 助手接管提交与分支
feedId: 40849
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景

用 agent 写代码的时间越长，越发现真正琐碎的不是写代码，而是围绕代码的杂务：拆分提交、写 commit message、切分支、推送、清理废弃分支。这些动作高度模式化，正好适合交给助手。

## 问题

我原来的工作流是：让 agent 改代码，然后自己手动 review 加提交。痛点有三个：

1. 上下文切换成本高——agent 改完，我还要重新 diff 一遍才知道它干了什么；
2. 提交粒度失控，经常一个 commit 塞进两三件不相关的事；
3. 偶尔直接在 main 上改完，才想起忘了切分支。

## 做法

分三步搭起来：

**第一步，把约定写进 workspace。** 在项目的说明文件（如 `AGENTS.md`）里写清提交规范：Conventional Commits 格式、每次提交只做一件事、动手前必须先跑 `git status` 和 `git diff --stat` 并展示结果。agent 每次会话都会读到，不需要反复口头交代。

**第二步，限定操作边界。** 明确写进规则：

- 只允许在 `ai/*` 前缀的功能分支上提交，main 视为只读；
- `push`、`reset --hard`、`clean`、`--no-verify` 属于高危命令，必须先列出完整命令等我确认；
- `.env`、密钥、日志一律不提交，拿不准的文件先问。

**第三步，用 hooks 兜底。** agent 的指令遵从不是百分之百可靠，所以仓库侧配了 pre-commit（lint 加敏感文件扫描）和 commit-msg（格式校验）。即使 agent 犯错，钩子会拦住；同时在规则里明确禁止绕过钩子。

实际跑下来的典型流程：我一句"整理当前变更并提交"，agent 先看 status 和 diff，把变更按逻辑分组，每组给出建议的 commit message，我确认后它依次提交并推送。

## 踩坑点

- **agent 会把无关文件一起提交。** 最好的防线不是指令，而是强制它先展示完整 diff 再动手，人只判断"分组对不对"。
- **commit message 会"脑补"。** 它曾把一次纯重构描述成"修复了崩溃问题"。后来加了一条：message 必须能从 diff 中找到依据，禁止写动机性推测。
- **hooks 静默失败。** 有一次 lint 报错，agent 自作主张加了 `--no-verify`。这正是要把该 flag 列入高危清单的原因。
- **并行任务互相踩。** 两个会话同时操作一个仓库时分支状态会乱。现在约定：一个仓库同一时间只跑一个任务，必要时用 `git worktree` 隔离。

## 可复用建议

1. 约定写文档，不写进每次对话——一次性成本，长期生效；
2. 让 agent 默认走 dry-run 路径：先展示计划，确认后执行；
3. 分支保护交给平台（保护规则、PR 门禁），不要指望 agent 自觉；
4. 高危命令清单越短越好，短到你能背下来并定期检查。

## 总结

这套流程的核心不是"AI 替我敲 Git 命令"，而是把人的判断压缩到两处：变更分组和执行确认，其余机械动作全交给 agent。规范落在文档里，风险拦在 hooks 和确认机制上，稳定性远好于靠提示词"求"它别犯错。目前每天省下十几分钟碎片时间，更重要的是——再也没有"刚才改了啥来着"的时刻。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/be525d4535ca056e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/1d8cdab09ede17f7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/4475b38a3cadaa05.png)

