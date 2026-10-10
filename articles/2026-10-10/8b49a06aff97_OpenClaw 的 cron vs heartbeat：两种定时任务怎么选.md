---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 41103
source: 综合讨论
publishedAt: 2026-10-10
---

## 背景

OpenClaw 里做自动化触发，主要就两条路：

- **cron 调度器**：标准 cron 表达式，到点把一条写好的消息投递给 agent，跟当前对话无关；
- **heartbeat**：网关按固定间隔把 agent 叫醒，让它读工作区里的 `HEARTBEAT.md`，结合当前状态自己判断"有没有事要做"。

两条路都能叫"定时"，但语义完全不同，混着用是新手最常见的翻车点。

## 问题

社区里反复出现的两类提问：

1. "我要每天早上九点推一份日报，用哪个？"——结果写进了 heartbeat，agent 被迫在主会话里干重活；
2. "我想让它有空就看看收件箱有没有异常"——结果拆成了十几条 cron，改一个检查逻辑要动一排 job。

本质上是没想清楚：**触发条件是时间，还是状态？**

## 做法

**1. 定点、明确结果的任务走 cron**

用 cron 工具添加 job：表达式 + 消息 + 会话策略。重活建议用 isolated session，避免撑爆主会话；想推送到频道，记得开投递（announce/delivery），否则结果只躺在日志里。也可以直接编辑 cron 的 `jobs.json` 批量管理。

**2. 巡检类任务走 heartbeat**

在 workspace 的 `HEARTBEAT.md` 写 checklist，一行一条，写清"什么条件下才需要叫我"，例如"如果有未处理的订阅消息，摘要后通知我"。agent 每个周期读一遍，没事就回一个简短 ack。

**3. 频率分开调**

heartbeat 间隔在网关配置里调（默认约 30 分钟），巡检够用即可；高频固定间隔（比如每 5 分钟）一律用 cron。

**4. 验证方式**

cron 看 job 运行日志和下次触发时间；heartbeat 改完 `HEARTBEAT.md` 后观察一两个周期，确认 agent 的 ack 行为符合预期。

## 踩坑点

- **时区**：cron 表达式默认按网关所在时区解析。服务器是 UTC 时，"每天 9 点"实际是北京时间 17 点，务必显式指定时区。
- **跑了但没消息**：没开投递的 cron job，日志里有结果、频道里静悄悄，别以为没执行。
- **heartbeat 不是后台队列**：它跑在主会话里，重任务会污染上下文、打断你正在进行的对话。重活挪去 cron 的 isolated session。
- **HEARTBEAT.md 堆积**：已完成条目不删，agent 每个周期都要重新评估，白烧 token。做完就删。
- **宕机不补跑**：网关重启期间 cron 错过就是错过，对时间敏感的任务要有兜底，比如加一条低频心跳巡检。
- **心跳间隔别贪快**：设成 1 分钟不会更快，只会让 ack 流量体现在账单上。

## 可复用建议

- 一句话判断标准：**知道"几点做"用 cron，只知道"要检查什么"用 heartbeat**。
- 把 heartbeat 当"值班巡检"，cron 当"排班表"：日报、备份、定点抓取走 cron；"有没有新东西、有没有异常"走 heartbeat。
- `HEARTBEAT.md` 控制在 3~5 行以内，条件写得越明确，误触发越少。
- 一个 job 一个职责，命名带用途前缀，方便日后批量清理。

## 总结

cron 和 heartbeat 不是替代关系：cron 负责确定性，heartbeat 负责主动性。先回答"这个任务的触发条件是时间还是状态"，再决定写在哪个文件里，绝大多数定时需求都能干净落地。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/e0680612bebd603b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/f3caa54e28484d8c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/c6e61f76844d484f.png)

