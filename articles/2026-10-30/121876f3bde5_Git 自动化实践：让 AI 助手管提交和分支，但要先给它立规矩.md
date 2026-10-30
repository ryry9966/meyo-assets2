---
title: Git 自动化实践：让 AI 助手管提交和分支，但要先给它立规矩
feedId: 43011
source: 综合讨论
publishedAt: 2026-10-30
---

## 背景

日常开发里不少时间耗在 Git 杂活上：写 commit message、拆分改动、清理过期分支、汇总 changelog。这些事重复、机械、套路固定，很适合交给常驻的 AI 助手。OpenClaw 的优势在于能定时跑、能接 MCP 工具、能在聊天里逐条跟你确认，于是我把仓库的提交与分支治理部分交给了它。跑了一个多月，把做法和踩过的坑记下来。

## 问题

直接说“帮我 commit"，常见结局有几种：一把梭 `git add .`，把 .env、调试脚本、构建产物全带进去；生成的 message 与真实 diff 对不上，甚至幻觉出不存在的功能；清理分支时误删同事还在用的分支；仓库处于 rebase 中间态时 agent 插手，给出完全不可信的建议。

核心矛盾：Git 是低容错、强状态的操作，而 agent 默认是“尽力完成”的行事风格。不做约束直接放开，出事只是时间问题。

## 做法

我的原则是两段式：agent 负责“看和提议”，写操作必须经人确认。拆成四步：

1. **限定能力边界**。skill 里写明子命令白名单：`status` / `log` / `diff` / `branch -a` 等只读命令可自主执行；`add` / `commit` / `push` / `branch -D` / `rebase` 一律先输出计划等确认，相当于最小权限集。

2. **提交流程标准化**。固定顺序：先跑 `git status` 和 `git diff --staged`，逐文件列出改动范围（禁止 `git add .`），基于真实 diff 生成 Conventional Commits 风格的 message，最后展示“文件清单 + message + 完整命令”三件套，确认后才执行，一次只做一个 commit。

3. **分支治理定时化**。每天早上跑一份只读报告：`git branch --merged main` 列出可安全删除的本地分支、两周以上无提交的远端分支、未合并的 WIP。删除动作永远手动执行。保护名单写死：main、dev、release/*、hotfix/* 不在清理范围。

4. **动作全量留痕**。agent 触发的每条 git 命令带时间戳写入日志，出问题可回溯。

skill 里的核心约束大概长这样：

```markdown
## Git 操作约束
- 写操作（add/commit/push/branch -D）先展示完整命令，等用户确认
- 禁止 git add .，必须逐文件确认
- push 只允许 --force-with-lease，禁止 --force
- message 只能基于 git diff --staged 的真实内容生成
- 清理分支仅限 --merged 结果，且不在保护名单内
```

## 踩坑点

- **`git add .` 是默认冲动**。第一次就差点把本地 .env 提交上去，把禁令写成硬规则并放进示例后才稳住。
- **幻觉式 message**：会脑补 diff 里没有的“优化了性能”。要求 message 中每个改动点都能在 diff 里找到对应行，提交前我也会对照扫一眼。
- **冲突中间态不可信**：仓库处于 rebase/merge 冲突时，agent 只允许输出“有 X 个文件待解决”，其余交给人工。
- **定时任务与手动操作撞车**：agent 跑报告时我正在 IDE 里切分支，读到脏状态。加了互斥检查：任务启动前确认没有进行中的 git 进程。

## 可复用建议

- 先放只读再放写：让 agent 先当两周“报告员”，确认判断质量后再开放提交通道。
- 确认粒度到命令级，不到任务级。“帮我整理分支”这种授权太粗，要看它将执行的每一条命令。
- 约束写进 skill 文件，别靠聊天口头约定——口头约定隔几天就会“失忆”。
- 保护名单宁多勿少，凡是有协作性质的命名分支都不让 agent 碰。

## 总结

AI 管 Git 的价值不在全自动，而在把机械劳动压缩成一次确认。一个多月下来的体感：commit message 基本不用自己写，分支清理从“想起来才做”变成每天有报告，而 push 和删分支这类高危动作仍握在自己手里。把 agent 当成手脚快但需要审批的实习生来用，这套流程才能长期稳定跑下去。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/663138e458f9b6e3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/475872acecd06a1a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/a65ce992b2a3c96d.png)

