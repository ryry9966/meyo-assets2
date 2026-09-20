---
title: 让 OpenClaw 帮你管 Git：提交与分支自动化实践
feedId: 38261
source: 综合讨论
publishedAt: 2026-09-20
---

## 背景

我同时维护着十来个仓库，日常工作里最容易失守的不是写代码，而是提交纪律：改完随手一个 `fix`，临时分支开完不删，两周后自己都分不清哪次改动对应哪个需求。OpenClaw 这类常驻本地的 Agent，恰好适合接手这种重复但有明确规则的活。

## 问题边界

先说清楚：让 AI 管 Git，不是让它替你做技术决策，而是三件事——

1. **提交信息**：基于真实 diff 生成符合 Conventional Commits 的 message；
2. **分支卫生**：识别陈旧分支、已合并分支，给出清理候选；
3. **状态巡检**：定期汇报哪些仓库有未提交改动、哪些与远端脱节。

所有写操作必须人工确认，这是前提，不是可选项。

## 做法

**第一步：权限收敛。** 给 Agent 建受限执行环境：exec 工具只允许在 `~/workspace` 下运行，git 命令走白名单（`status`/`diff`/`log`/`branch`/`commit`），显式禁掉 `push --force`、`reset --hard`、`clean -fd`。MCP 的 git server 天然比裸 shell 安全但灵活性差，我用的是 shell + 白名单的组合。

**第二步：仓库级约定。** 每个仓库放一份 `CONVENTIONS.md`，写明提交前缀、message 语言、禁止提交的路径。Agent 提交前先读它，比在全局提示词里堆规则有效得多。

**第三步：定义 `/commit` 流程。** 先 `git status` + `git diff --stat` 做概览，再对变更文件取局部 diff，生成 message 后把待执行命令回显给我，确认后才落盘。

**第四步：定时巡检。** 用 cron 每晚触发一次会话，汇总未提交、未推送的仓库，列出已合并分支的删除候选清单，我来拍板。

## 踩坑点

- **只给 `git status` 不给 diff**，生成的 message 全是编的。必须喂真实 diff。
- **大仓库 diff 撑爆上下文**。先 `--stat` 再按文件取，别一次全量灌进去。
- **Agent 热心过头**，把临时脚本一起 `git add .`。我改成只允许 `add <file>`，并在约定文件里重申 ignore 规则。
- **分支清理误删**。过滤逻辑固定用 `git branch --merged`，删除前展示最近提交时间让我确认。
- **message 语言漂移**，用一周就混了。最后在约定文件里钉死：英文 type + 中文 subject。

## 可复用建议

- 读操作全自动，写操作半自动，破坏性操作必须确认；
- 约束做两层：提示词说清楚 + 工具白名单兜底，只靠 prompt 不可靠；
- 先只跑只读巡检两周，确认 Agent 的判断和你预期一致，再放开提交权限；
- 每个仓库一份约定文件，Agent 会自己读，规则跟随仓库走，换机器也不丢。

## 总结

这套流程的价值不在"自动化"本身，而是把好的提交习惯的执行成本降到接近零。Agent 不会替你写出更好的代码，但它能让每个仓库的历史保持可读——半年后回溯问题时，这比什么都值钱。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/f04437084b31b71b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/86cebd925f071cf7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/1cd55f6671a2fb0a.png)

