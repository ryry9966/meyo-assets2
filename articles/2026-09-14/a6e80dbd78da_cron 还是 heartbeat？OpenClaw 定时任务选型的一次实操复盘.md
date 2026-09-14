---
title: cron 还是 heartbeat？OpenClaw 定时任务选型的一次实操复盘
feedId: 37514
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

OpenClaw 里有两种让 agent「周期性干活」的机制：**cron jobs** 和 **heartbeat**。刚上手时很容易混淆——两者都能让 agent 定期醒来做事，但设计目标完全不同。最近我把手上几个自动化场景在两者之间迁来迁去折腾了一轮，记录一下选型思路。

## 两种机制的本质区别

- **cron**：确定性调度。到点必触发，支持 cron 表达式、时区配置、独立会话（isolated session）和任务持久化。本质是「调度器 + 单次任务执行」。
- **heartbeat**：agent 周期性自省。gateway 按间隔唤醒主会话，agent 读取 HEARTBEAT.md 里的检查清单，自行判断这次有没有事可做，没事就快速返回结束。本质是「定时巡逻 + 自主决策」。

一句话总结：cron 是「几点必须做什么」，heartbeat 是「每隔一阵看看有没有要做的」。

## 做法：按确定性分层

我目前的分层规则：

1. **时间点刚性 → cron**。日报 9:00 发出、每 2 小时抓一次 RSS、每周一清理临时目录。用 cron 表达式挂上，重活配独立会话避免污染主上下文，结果通过 delivery 推回主会话或通知渠道。
2. **态势感知类 → heartbeat**。比如「收件箱里有需要处理的吗」「部署目录有没有异常文件」。这类任务没有明确时间点，靠 agent 自己判断值不值得动手。
3. **混合模式**。cron 负责采集（拉数据、落盘），heartbeat 巡检时看结果做决策，比全塞进 heartbeat 稳定得多。

heartbeat 调优三板斧：HEARTBEAT.md 只放检查项，一句话一条；用间隔控制频率；配置活跃时段（比如只在工作时间唤醒），成本立竿见影。

## 踩坑点

- **HEARTBEAT.md 写太长**。每次心跳都消耗 context 和 token，清单从 10 行涨到 40 行后，一晚上「闲巡逻」烧掉的 token 比正事还多。现在控制在 5 行以内，明确任务全部挪进 cron。
- **把心跳当精确调度器**。间隔只是「约」，负载高时会漂移，千万别用来做「9:00 整必须发出」的事。
- **cron 独立会话的隔离性是双刃剑**。任务在隔离会话跑完，主会话并不天然知道结果。忘配 delivery 会出现「任务明明跑了但没人收到」的幽灵现象。
- **时区**。cron 调度时区和本机不一致时，日报会准时出现在错误的钟点。建任务时显式声明时区。
- **重任务卡心跳**。在心跳里塞了一个跑 10 分钟的检查，期间后续 beat 全在排队，外在表现就是「心跳失联」，排查了半天才发现是任务阻塞。

## 可复用建议

- 决策一句话：**有确切时间约束选 cron，只有「经常看看」的诉求选 heartbeat**。
- 重活一律 cron + 独立会话；heartbeat 保持轻量，只做判断、不做重执行。
- 定期审计 heartbeat 的 token 开销，它是唯一「闲着也花钱」的机制。
- cron 任务要有生命周期管理：一次性任务跑完就删，别让任务列表变成考古现场。

## 总结

两者不是替代关系。cron 给你确定性，heartbeat 给你自主性。把「必须准点」的事交给 cron，把「帮忙盯着」的事交给 heartbeat，再控制好心跳成本，OpenClaw 的定时自动化基本就稳了。如果你有更复杂的编排需求，欢迎在评论区交流踩坑经验。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/85bed616cdfb0c27.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/1a813c96a95c7e15.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/8e77eac7828824f3.png)

