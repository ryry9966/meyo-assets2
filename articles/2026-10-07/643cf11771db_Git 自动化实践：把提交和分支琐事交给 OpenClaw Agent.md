---
title: Git 自动化实践：把提交和分支琐事交给 OpenClaw Agent
feedId: 40753
source: 综合讨论
publishedAt: 2026-10-07
---

## 背景

我平时拿 OpenClaw 当个人工程助理，接了 shell 工具和几个 MCP server 之后，开始把日常杂活逐步往 agent 那边挪。Git 是最先被啃下来的：写 commit message、切分支、清理废弃分支、汇总各仓库动态——这些事琐碎、重复，但规则清晰，天然适合交给 agent。

## 问题

先列痛点：

- commit message 质量忽高忽低，改了一天代码后根本懒得好好写；
- 多任务并行时经常忘切分支，WIP 直接提交到了 main；
- 三个月下来本地堆了四五十个已合并分支，没人清理；
- 想回顾"这周各仓库都动了什么"，得手动挨个 `git log`。

## 做法

环境：OpenClaw + shell 工具（git MCP server 亦可），只授权两个工作仓库目录。

**第一步，定规矩。** 在仓库根目录放一份给 agent 读的说明文件：commit 遵循 Conventional Commits、分支命名 `feature/xxx`、main 分支上 agent 只允许生成报告，不允许提交。

**第二步，拆成两个流程。**

- *提交流程*：我 stage 完改动后说一句"整理提交"。agent 先跑 `git diff --staged --stat`，再分文件看 diff，归纳改动、生成带 scope 的 message，展示给我确认后才执行 `git commit`。push 始终由我手动做。
- *巡检流程*：用 OpenClaw 的定时任务每天早上跑一次：列出未合并分支、检查 main 是否有未 push 的提交、对已合并超过两周的本地分支给出清理建议——只建议，不执行删除。

**第三步，权限收口。** 给 git 命令做白名单：`status / diff / log / branch / commit / stash` 放行；`reset --hard`、`push`、`rebase`、`clean` 一律拒绝，需要时我手动跑或临时解锁。

## 踩坑点

1. **Message 幻觉**：早期 agent 会根据文件名脑补改动内容，message 和实际 diff 对不上。后来强制它先引用 diff 里的关键 hunk 再写 message，准确率明显上来了。
2. **大 diff 撑爆上下文**：一次几千行的重构 diff 直接把会话搞崩。改成先看 `--stat`，再挑重点文件看局部 diff。
3. **pre-commit 钩子循环**：格式化钩子改了文件，agent 困惑后反复重试 commit。现在遇到钩子修改文件就让它停下交还给我。
4. **凭证安全**：绝对不要把 token 贴进对话。走系统 ssh-agent 和 credential helper，agent 只调用 git 本身。

## 可复用建议

- **先只读后写入**：让 agent 先跑两周 `status/diff/log`，观察它对仓库的理解程度，再开放 commit 权限。
- **规范写成文件放进仓库**，比每次在对话里口头强调可靠得多。
- **破坏性命令永不进白名单**，宁可麻烦一点。
- **留操作日志**：agent 执行过的每条 git 命令都记下来，出问题能回溯。
- **不可逆操作保持人工确认**：push、删分支这类动作别为了省事自动化。

## 总结

这套东西跑了一个多月，commit message 规范率接近 100%，废弃分支不再堆积。核心结论是：agent 适合接管有明确规则的"体力活"，而涉及判断和不可逆动作的环节，保留人工节点反而让整套流程更敢用。OpenClaw 这类框架的价值就在于此——工具接入便宜，边界由你自己画。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/aca7236ed13d6663.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/6782362ebca3a794.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/ab12b68fd012d1e1.png)

