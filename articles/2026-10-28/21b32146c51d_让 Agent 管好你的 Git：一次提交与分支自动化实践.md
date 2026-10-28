---
title: 让 Agent 管好你的 Git：一次提交与分支自动化实践
feedId: 42817
source: 综合讨论
publishedAt: 2026-10-28
---

## 背景

每天写代码，真正花在 Git 上的时间是碎片化的：想一个合适的 commit message、清理本地堆了几十天的分支、确认哪些改动忘了 push。单件不超过两分钟，加起来却很磨人。OpenClaw 的 agent 天然具备 shell 执行与定时任务能力，很适合接手这类"低风险、高频次"的操作。

## 问题

把 Git 交给 AI 助手，核心矛盾是：**能安全自动化的往往不值得自动化，值得自动化的往往有风险**。生成 commit message 是安全的，但 force push、rebase、删分支属于破坏性操作。所以先划一条清楚的线：

- 读操作与分析类：全自动
- 写操作：半自动，agent 提议、人确认
- 破坏性操作：原则上禁止 agent 直接执行

## 做法

1. **准备环境**：给 agent 的 workspace 单独配置 git 身份和凭据，不要复用个人全局 SSH key，用 per-repo deploy key 做最小授权。
2. **写一个 git-helper skill**，三个入口脚本：
   - `summarize`：读 `git diff --staged`，按 Conventional Commits 生成候选 message（只喂真实 diff，不给它自由发挥的空间）；
   - `status-report`：输出未提交文件、本地领先/落后的提交数、未推送分支；
   - `branch-cleanup --dry-run`：列出 `git branch --merged` 中的本地分支，仅标注可删项。
3. **接入执行工具**：通过 MCP 或内置 exec 暴露这三个脚本，配置里做命令白名单，确保 `push -f`、`reset --hard` 永远不出现在自动执行路径上。
4. **加定时任务**：每天早上把 status-report 推到私人频道；每周五跑一次 branch-cleanup 的 dry-run 报告，删不删由人决定。

## 踩坑点

- **message 幻觉**：早期我把整个文件内容都传给模型，它会把没改的逻辑也写进 commit message。后来严格只传 diff，输出质量立刻稳定。
- **dry-run 不能省**：branch-cleanup 第一版直接删了 `--merged` 列表里的分支，结果在某个用 squash merge 的仓库里把未合并的工作删了——squash 之后分支并不算 merged。现在改成"报告 + 人工确认"。
- **并发冲突**：agent 跑写操作时我正好在另一个终端操作同一仓库，撞了 index.lock。现在 agent 执行写操作前会先检查锁文件，有冲突就跳过并上报。
- **不要让 agent 自动 push 到共享分支**：即使内容正确，也不该绕过 review 流程。

## 可复用建议

- skill 保持"小脚本 + 明确参数"，不要让模型自由拼装 git 命令；
- 所有自动化输出**报告**而不是**静默改动**，你随时能审计它做了什么；
- skill 脚本放进仓库做版本管理，改坏了能回滚；
- 破坏性操作一律设计成"agent 准备好命令，人复制执行"。

## 总结

Git 自动化追求的不是全自动，而是把 agent 当成一个不知疲倦的助理：它负责观察、汇总、起草，你负责决策和签字。这条边界守住之后，每天省下的十几分钟是真的，而且不用担心哪天醒来发现仓库被它"整理"没了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/39faab6e71ac4859.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/f9a8d1fbeff674c0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/852b874fc283d4f5.png)

