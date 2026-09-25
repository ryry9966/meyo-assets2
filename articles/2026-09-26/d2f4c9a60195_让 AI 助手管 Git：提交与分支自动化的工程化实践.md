---
title: 让 AI 助手管 Git：提交与分支自动化的工程化实践
feedId: 39044
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

写代码时最琐碎的往往不是写代码本身，而是收尾：拆分暂存区、写 commit message、清理已合并的分支。这些事简单、重复、规则明确，正好适合丢给跑在 OpenClaw 里的 agent。我折腾了两周，把「提交整理」和「分支清理」两个流程交给了助手，稳定运行一段时间后，把做法和教训记录下来。

## 问题

- 本地改动攒到深夜，一次 commit 塞了三件事，回溯时完全想不起哪次改了什么；
- 已合并分支越积越多，`git branch` 一屏都放不下；
- 让 AI 直接操作 Git 本身有风险：误 `git add -A` 把密钥提交进去、自动 push 到远端、自作主张 rebase。

## 做法

1. **权限在工具层收敛**。我没有让 agent 裸调 shell，而是把 Git 封装成一个 MCP 工具，白名单只放行 `status / diff / log / add / commit / branch -d / stash`。`push`、`reset --hard`、`rebase` 一律不进白名单，prompt 里怎么叮嘱都不如工具层直接不给。
2. **提交前必须读 diff**。约定 agent 先 `git status` 加 `git diff --staged`，基于实际改动写 Conventional Commits 风格的 message；多组逻辑改动拆成多次原子提交，不允许一把梭。
3. **分支清理走确认制**。定时任务扫描已合并到主干的本地分支，输出列表给我确认后再删；主干分支和任何未合并分支是硬排除项，写死在 skill 里。
4. **默认只动本地**。所有操作止步于 commit；push 需要我口头确认后手动触发，且永远不使用 `--force`。

## 踩坑

- **敏感文件差点入库**。最开始 agent 会把 `.env` 一起 add，好在发现得早。之后强制要求：每次 add 前先输出暂存文件清单，命中敏感路径直接终止任务。
- **只看文件名写 message 会幻觉**。有一次 message 写的是"重构配置模块"，diff 里其实是改了测试。现在规定不读完整 diff 不许动笔。
- **和我的编辑撞车**。我正在改文件时自动任务跑了，把半成品 commit 了进去。补了空闲检测：工作区有未暂存的活跃改动、或最近几分钟检测到编辑行为，就跳过本轮。
- **合并冲突不要让 AI 解决**。让 agent 自动处理过一次冲突，结果把两边的逻辑都改了。改成遇到冲突立即停下、报告冲突文件，人来裁决。

## 可复用建议

- 把 Git 操作封装成独立的 skill 或 MCP 工具，权限收敛在工具层，比在 system prompt 里"提醒"可靠得多；
- 关键动作前输出 dry-run 预览（要 commit 哪些文件、用什么 message），确认后再执行；
- 保留完整操作日志，出问题能回放；
- 宁可多个小 commit，也不要攒一个大 commit——这是自动化最容易帮倒忙的地方；
- 自动化程度逐步放开：先只读出报告，再本地 commit，最后经确认 push，每档观察一周再升一级。

## 总结

AI 助手管 Git 的价值不在"全自动"，而在把机械环节抽走、把判断留给人。权限白名单、默认本地、强制读 diff——这三条护栏立住，流程就能长期跑下去。与其追求一键全托管，不如先让它每天帮你把工作区收拾干净，这已经能省下不少心智负担。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/7725201ae3a846bc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/dd7c1aabc7cf0325.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/9c6c04298e7168a9.png)

