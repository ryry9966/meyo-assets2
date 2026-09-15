---
title: cron 还是 heartbeat？OpenClaw 定时任务的选型笔记
feedId: 37695
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

OpenClaw 的 agent 有两条"自己醒过来干活"的路径：内置的 heartbeat（心跳循环）和外置的 cron 调度。两者都能让 agent 周期性执行 prompt，但设计目标完全不同。社区里反复出现的问题不是"不会配"，而是"配错了地方"——该走 cron 的挂在了 heartbeat 上，反过来也一样。

## 问题在哪

两个典型症状：

1. **用 heartbeat 做早报**：指望 agent"到点就发"。实际时间漂移严重（心跳会等主会话空闲才执行），主 session 上下文被巡检内容塞满，token 消耗直线上升。
2. **用 cron 做环境巡检**：每 5 分钟开一个隔离 session 去查状态，跑一天留下几百条 session 记录，而且每个 session 都是"失忆"的，读不到主对话里的任何上下文。

## 两者的本质区别

- **heartbeat**：单循环、共享主 session、按间隔触发、agent 自主决定是否行动。每次醒来先读 HEARTBEAT.md，没事就静默返回，有事才真正执行。它是"环境感知层"。
- **cron**：独立调度器、默认隔离 session、支持 cron 表达式和一次性 at 任务、可把结果投递到指定 channel。它是"任务调度层"。

## 我的选型步骤

1. 先问三个问题：**需要准点吗？需要干净上下文吗？需要主动投递结果吗？**
2. 三个都是 → cron。日报、备份、监控摘要、一次性提醒全走 cron：用 `openclaw cron add` 创建，schedule 支持 cron 表达式或 every 间隔；session 保持默认隔离，prompt 里把上下文写全，不要指望"上文"；结果配置投递到目标 channel。
3. 只需要模糊周期、且依赖主对话语境的轻量巡检 → heartbeat。比如"顺手看看有没有没处理的 TODO"。事项写进 HEARTBEAT.md，每条尽量一句话，降低每次心跳的判断成本。
4. 兜底原则：能写成确定性任务的，一律迁到 cron；HEARTBEAT.md 只留真正的"氛围级"事项。

## 踩坑点

- **心跳不保证准点**：主会话正在跑别的 turn 时，心跳会被跳过。别拿它做"整点必须发生"的事。
- **token 成本**：heartbeat 每次唤醒都有开销，哪怕 agent 最后选择静默。HEARTBEAT.md 越长越贵。
- **时区**：容器里 cron 默认 UTC，"早上 8 点的日报"发到了下午。创建任务时显式指定时区。
- **不补跑**：gateway 停机期间错过的 cron 任务不会自动补跑，at 类一次性任务错过就没了。时间敏感的任务要配告警，不能盲信。
- **重复触发**：同一件事 heartbeat 和 cron 都配过、忘了删旧的，用户收到两份日报。加新任务前先 `openclaw cron list` 盘一遍。
- **排查路径**：`cron runs`（按任务 id）能看到每次执行的状态和耗时，报障先看这里，别只翻聊天记录。

## 可复用建议

- 口诀：**准点交付用 cron，环境感知用 heartbeat**。二者是分工，不是替代。
- heartbeat 保持极简：HEARTBEAT.md 控制在三五行，间隔宁长勿短。
- cron 任务保持幂等：prompt 里写清楚"若上次已完成则跳过"，降低重复执行的危害。
- 季度性审视：一次性 at 任务执行完即失效，周期任务要定期清理，删掉已经没人看的日报。

## 总结

heartbeat 是 agent 的"脉搏"，负责低成本的持续感知；cron 是外挂的"闹钟"，负责精确、隔离、可投递的任务执行。下次加定时能力前，用"准点 / 上下文 / 投递"三问过一遍——九成场景答案都指向 cron，剩下的一成才轮到 heartbeat。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/fba586899b39e788.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/90da6767697738d6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/43bd91145622f2cc.png)

