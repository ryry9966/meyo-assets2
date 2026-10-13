---
title: cron 还是 heartbeat？OpenClaw 定时任务的选型笔记
feedId: 41444
source: 综合讨论
publishedAt: 2026-10-13
---

# 背景

OpenClaw 里有两条"让 agent 定时干活"的路：

- **cron**：传统意义上的定时任务。你给出 cron 表达式和一段 payload，到点就把这段话喂给 agent 执行。
- **heartbeat**：agent 主循环自带的心跳。每隔一段时间（默认 30 分钟）agent 醒一次，读工作区里的 `HEARTBEAT.md`，自己判断有没有事要做，没事就继续睡。

很多人的第一个自动化是从改 `HEARTBEAT.md` 开始的，用着用着会发现它和 cron 职责重叠，很容易配出"两套机制干同一件事"的冗余。

# 问题：触发条件是"时间"还是"状态"

选型前先问自己一句话：这个任务的触发条件，是**时间到了**，还是**状态变了**？

- "每天早上 9 点给我一份日报摘要"——时间到了就做，用 **cron**。
- "盯着这几个订阅源，有值得说的再告诉我"——不知道何时有变化，用 **heartbeat**。

再补两个判断维度：

1. **精确性**：cron 表达式是准点触发；heartbeat 的间隔只是近似，上一跳的回合没跑完，下一跳会顺延。凡是"必须在某时刻发生"的事，别指望心跳。
2. **上下文**：heartbeat 跑在主会话里，天然带着完整对话上下文，适合"结合情况判断"的任务；cron 的投递更干净独立，适合可复现的固定产出。

# 落地步骤

1. 把需求写成一句话："每到 XX 时间做 YY" → cron；"随时检查 ZZ，有变化就 YY" → heartbeat。
2. cron 任务给出表达式（如 `0 9 * * *`）加一段明确的 payload：做什么、输出到哪个渠道、什么格式。一次性提醒用一次性任务（at 语义），别让 agent 自己"记住时间"。
3. `HEARTBEAT.md` 里只写巡检项，每条尽量写成可判断的问句，并注明"无事时跳过"，条目控制在 3~5 条以内。
4. 观察几天 token 消耗和响应及时性，再调整心跳间隔。
5. 两者可以组合：cron 负责"准点投递"，heartbeat 负责"兜底巡检"，各管一层。

# 踩坑点

- **时区**：cron 按 gateway 进程时区解释，Docker 里默认 UTC。"早上 9 点"跑成下午 5 点是最常见的翻车，显式设置 TZ。
- **heartbeat 写太满**：检查项含糊（比如"看看有什么新鲜的"），每一跳都会触发完整回复，token 消耗暴涨。每条都要有明确的"无事"判定。
- **拿心跳当秒级定时器**：心跳不是 tick，长回合会阻塞并顺延下一跳。
- **重复值班**：同一件事既在 cron 里又在 HEARTBEAT.md 里，用户收到两份早报。定期清理两边，保证一件事只有一个 owner。

# 可复用建议

- 分层默认值：确定性时间任务 → cron；状态巡检 → heartbeat；临时提醒 → 一次性 cron。
- cron 的 payload 里显式写清输出渠道和格式，别留自由发挥的空间。
- 新任务先低频跑一两天，看消息量和消耗再调频率，别一上来就配密集调度。

# 总结

cron 是闹钟，heartbeat 是值班。到点的事交给闹钟，需要判断的事交给值班。两者可以共存，但别让它们值同一个班。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/172c8eafec68769c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/53804de6cf3143df.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/7a8efc5060c0dcc4.png)

