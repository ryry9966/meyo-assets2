---
title: cron vs heartbeat：OpenClaw 定时任务的选型与踩坑
feedId: 39136
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

OpenClaw 里让 agent「定时干活」有两条路：**cron 任务**和 **heartbeat**。新用户最常见的困惑是"该用哪个"，更常见的错误是两个都上，或者全塞给 heartbeat，结果不是烧 token 就是漏任务。这篇按实际使用经验梳理一下取舍。

## 两者的本质区别

**cron 是闹钟**：时间到了就触发，把写好的 prompt 交给 agent 完整执行一次。确定性调度，适合"几点必须做什么"。

**heartbeat 是巡场**：agent 按固定间隔（默认半小时左右）醒来，读一遍 HEARTBEAT.md，自己判断有没有事要做，没事就应该跳过。它本质是**条件触发**，只是借了周期性的壳。

一句话：时间确定 → cron；只有条件、没有确定时刻 → heartbeat。

## 具体做法

1. **先盘点任务**。逐条问："这件事有明确的执行时刻吗？"每日早报、周报汇总、定时备份提醒——有，归 cron。"帮我盯着邮箱，有重要的再告诉我"，没有，归 heartbeat。
2. **固定时刻用 cron**。写 prompt 时当它是独立任务：做什么、输出写到哪、失败怎么处理，写全。表达式和时区以 `openclaw cron --help` 与官方文档为准。
3. **弹性巡检写进 HEARTBEAT.md**。每条一行祈使句，**必须带跳过条件**，例如"检查 XX，若无新增则直接跳过，不输出任何内容"。
4. **心跳间隔先调大**。巡检类任务 30 分钟一次通常过于频繁，先调到能接受的最低频率，观察一周再收紧。
5. **组合时划清边界**：cron 管交付，heartbeat 管巡检，同一件事不要两边都配。

## 踩坑点

- **heartbeat 烧 token**。HEARTBEAT.md 写得含糊，agent 每次醒来都"认真干活"，一天几十次醒来就是几十次开销。跳过规则写清楚能省一大半。
- **容器时区**。Docker 里跑 Gateway 默认可能是 UTC，cron 表达式按本地时间写就差 8 小时。先确认容器 TZ 再定表达式。
- **cron prompt 写成巡检项**。cron 触发就是一次完整执行，prompt 里写"看看有没有要做的"会导致输出不可控，必须写成完整指令。
- **漂移预期**。heartbeat 不是精确调度，重启或忙碌时会顺延；精确到分钟的提醒必须走 cron。
- **重复触发**。同一任务两边都配，用户会收到两份消息。改 HEARTBEAT.md 时顺手 `cron list` 核对一遍。

## 可复用建议

- HEARTBEAT.md 保持短小，超过十行就该审视：哪些巡检项已经固化了？沉淀成 cron。
- cron 任务要有输出落点（写文件、发频道），方便事后审计，而不是只翻聊天记录。
- 心跳频率和 HEARTBEAT.md 一起调：任务写得越明确，频率就可以压得越低。

## 总结

cron 和 heartbeat 不是竞争关系，而是分层：**确定性时间驱动交给 cron，弹性条件驱动交给 heartbeat**。控制成本的关键不在选哪个，而在 HEARTBEAT.md 的跳过规则写得够不够狠。先分类任务，再选机制，最后用一两周执行日志校准，这套流程基本能避开定时任务里 90% 的坑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/4b82fd27eeea0561.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/688787c21461396e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/a310467f9271163c.png)

