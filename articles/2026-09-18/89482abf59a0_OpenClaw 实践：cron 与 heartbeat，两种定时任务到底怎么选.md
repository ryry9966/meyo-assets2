---
title: OpenClaw 实践：cron 与 heartbeat，两种定时任务到底怎么选
feedId: 38021
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

OpenClaw 网关里有两套"定时唤醒"机制，都跑在本地 gateway 进程里，不依赖外部调度器：

- **cron**：通过 `openclaw cron add` 或会话内 cron 工具创建的任务列表，落在 `~/.openclaw/cron/jobs.json`。到点把一段 prompt 注入会话执行，支持 one-shot 和循环，结果可以回投到指定消息通道。
- **heartbeat**：网关按 `agents.defaults.heartbeat.every` 的间隔，周期性给主 agent 发一个心跳。agent 读取工作区里的 `HEARTBEAT.md`，自己判断"现在有没有事要做"，没事就保持沉默。

两者看似都是"定时"，语义完全不同，混用是新人最常见的翻车点。

## 问题

什么该交给 cron，什么该交给 heartbeat？划分不清的后果是：任务重复执行、token 白白烧掉、半夜被 agent 的"一切正常"刷屏。

## 我的划分标准

一句话：**cron 管"定点交付"，heartbeat 管"条件守望"。**

适合 cron 的：
- 每个工作日早 9 点的工作简报
- 每周五生成周报并发到 Telegram
- "明早 8 点提醒我"这类一次性任务

特征：时间点是硬约束，结果默认要送达。

适合 heartbeat 的：
- 盯某个状态，有变化才说话（磁盘水位、PR 有没有新评论）
- 收件箱分诊：没新东西就闭嘴

特征：时间是软约束，"安静"才是正常输出。

### 配置步骤

**cron：**
1. `openclaw cron add --name daily-brief --cron "0 9 * * *" --prompt "……" --session isolated --announce`
2. 产报告类任务用 isolated 会话，别污染主会话上下文；提醒类用 `--at` 建 one-shot。
3. `openclaw cron list` 看任务，`openclaw cron runs --id <id>` 查执行历史，排障先看这里。

**heartbeat：**
1. `~/.openclaw/openclaw.json` 里把 `heartbeat.every` 设为 30–60 分钟起步。
2. `HEARTBEAT.md` 只写三件事：要盯的条件、满足时做什么、不满足时怎么办。控制长度。
3. 配 `activeHours` 限制夜间静默。

## 踩坑点

1. **时区**：容器里 TZ 是 UTC，`0 9` 实际是下午 5 点。要么给网关设 TZ，要么按 UTC 写表达式。
2. **isolated 会话没有聊天记忆**：cron 的 prompt 必须自包含，别指望它记得昨天聊过什么。heartbeat 反而跑在主会话里，能看到上下文——这本身就是一条选型依据。
3. **心跳有抖动**：间隔是"大约"，网关忙时会顺延，别拿 heartbeat 做分秒级的事。
4. **心跳成本**：每一拍都消耗 token。`HEARTBEAT.md` 写成三页文档，一晚就是几十次无效燃烧。条件复杂的场景，改成"heartbeat 发现可疑就临时注册一个 one-shot cron 做深查，查完删掉"。
5. **静默约定**：不明确写"没事别汇报"，agent 会每拍都发"一切正常"。
6. **重叠执行**：cron 和 heartbeat 同时盯一个状态可能重复动作，任务务必幂等。

## 可复用建议

- 决策口诀：**定时必达 → cron；有变才说 → heartbeat。**
- 心跳间隔从 60 分钟起调，观察一周噪声和账单再收紧。
- 让两者协作：heartbeat 当哨兵，发现异常时动态注册 cron 任务做一次性深挖。
- 所有定时任务输出落日志，`cron runs` 和网关日志是排障第一现场。

## 总结

cron 是闹钟，heartbeat 是保安巡逻。闹钟保证准点，巡逻保证异常不漏。选型的关键不在功能强弱，而在两个问题：**时间是不是硬约束？沉默能不能接受？**想清楚这两问，剩下的只是几行配置的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/94838e80e14f7cca.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/12ab9de26a68bc0d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/30cbccece6fdfceb.png)

