---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 41305
source: 综合讨论
publishedAt: 2026-10-12
---

## 背景

OpenClaw 里让 agent「自己动起来」有两个入口：**cron** 和 **heartbeat**。cron 是计划任务，按表达式、间隔或一次性时间点触发运行；heartbeat 是周期唤醒，默认每 30 分钟左右叫醒 agent 一次，让它读工作区里的 `HEARTBEAT.md`，判断有没有事要办。

两者最终都落在同一条链路上——给 agent 一个 prompt，跑完把结果投递到频道。但触发模型完全不同：cron 是时间驱动，heartbeat 是状态驱动。

## 问题

我见过两类典型误用：

- 用 heartbeat 实现「每天 9 点发日报」，结果触发时间漂移，有时 9:07，有时 9:40；
- 要「盯着某个条件、变了就处理」，却建了七八个 cron job，每个都去查同一个状态。

根因是没先问一句：这个任务是**必须在某个时刻发生**，还是**条件满足时才需要发生**？

## 做法

**第一步：分类。** 有明确时间点的（日报、定时备份、周会提醒）走 cron；描述成「帮我盯着 X」的（收件箱、CI 状态、待办清单）走 heartbeat。

**第二步：配 cron。** 直接让 agent 调 cron 工具创建任务，说清四件事：

1. schedule：cron 表达式 / 固定间隔 / 一次性 `at`；
2. 会话：一般用 isolated（独立新会话），确实需要主会话上下文才用 main；
3. payload：任务内容写全，隔离会话看不到聊天记录；
4. delivery：不投递、只发摘要、还是发完整结果，投到哪个频道。

建议先建一个几分钟后的 `at` 一次性任务试跑，确认投递路径没问题，再转成周期任务。

**第三步：配 heartbeat。** 做两件事：把「轮询清单」写进 `HEARTBEAT.md`，只列值得打扰你的条目，并明确「无事可做时回复 HEARTBEAT_OK」；用 `HEARTBEAT_INTERVAL_MS` 把间隔调到 15–30 分钟起。

## 踩坑点

- **token 成本**：heartbeat 每次唤醒都消耗 token，哪怕最后什么都没做。把间隔压到 5 分钟，账单会很难看。
- **准时性**：heartbeat 会漂移、可能被跳过，别拿它做任何「整点」语义的事。
- **上下文缺失**：cron 的 isolated 会话是干净的，payload 必须自包含——路径、ID、要做什么，一次写全。
- **时区**：cron 按宿主机本地时间解释表达式，Docker 容器默认 UTC，任务错位一小时是常见事故。
- **任务重叠**：心跳任务执行时间超过间隔时，下一轮会叠上来；cron 有单次运行超时，长任务要显式设置。

## 可复用建议

- 一句话判断法：**cron 当日历用，heartbeat 当轮询循环用**。
- heartbeat 清单宁少勿多，加一条兜底规则：「清单之外的事一律不动」。
- 周期性 cron 任务统一投递到一个专用「审计频道」，方便回查它到底跑没跑、跑了什么。
- 网关重启后留意缺失任务有没有被补跑，关键流程别依赖隐式补偿。

## 总结

cron 和 heartbeat 不是竞争关系，而是互补：前者解决「什么时候做」，后者解决「要不要做」。先判断任务是时间敏感还是状态敏感，再选机制，绝大多数误用会自然消失。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/0a1719a66e313242.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/fbdd5a70d2652a69.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/fff375d384b24f44.png)

