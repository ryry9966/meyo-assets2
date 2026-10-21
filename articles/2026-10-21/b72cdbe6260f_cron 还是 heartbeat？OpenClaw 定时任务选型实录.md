---
title: cron 还是 heartbeat？OpenClaw 定时任务选型实录
feedId: 42174
source: 综合讨论
publishedAt: 2026-10-21
---

## 背景

用 OpenClaw 跑个人助理一段时间后，几乎都会遇到同一个问题：想让 agent 主动做事，但系统里有两个入口——**cron** 和 **heartbeat**。两者都能让 agent"定时醒来"，但机制和成本结构完全不同。选错了，要么消息轰炸，要么关键提醒静默缺席。

## 问题

我最初的做法是把所有需求都塞进 cron：每小时查一次收件箱、工作日 9 点发晨报、任务到期提醒……一周攒出 20 多个 job。其中"每小时检查收件箱"这类巡逻型任务 token 开销大，且绝大多数轮次输出是"没有新消息"。反过来，用 heartbeat 做"9 点准点日报"又做不到——它是固定间隔的巡逻，不是时钟。

## 两种机制的本质区别

- **cron：时刻驱动。** 调度器精确触发，到点就把一条确定的消息作为 agent turn 注入会话，并可指定投递渠道。触发即执行，执行即产出。
- **heartbeat：事件驱动（带判断）。** gateway 按固定间隔唤醒 agent；agent 检查工作区上下文，认为没有值得汇报的事就返回静默标记，什么都不发。每次唤醒都是一次完整的 LLM 调用，"说不说"由模型判断。

一句话：cron 回答"这个时刻该做什么"，heartbeat 回答"现在有没有值得说的事"。

## 做法

**1. cron 承接确定时刻的任务。** 配置在 `openclaw.json` 的 `cron.jobs`（字段名以当前版本文档为准）：

```json
{
  "name": "morning-digest",
  "schedule": { "kind": "cron", "expr": "0 9 * * 1-5", "tz": "Asia/Shanghai" },
  "payload": { "kind": "agentTurn", "message": "读取工作区 TODO.md 和未读消息，输出不超过 10 行晨报" },
  "delivery": { "channel": "telegram", "to": "<chat-id>" }
}
```

**2. heartbeat 承接巡逻。** 在 `agents.defaults.heartbeat` 里设置间隔（毫秒，先从 30 分钟起），并在工作区 `HEARTBEAT.md` 写清楚"每次醒来检查什么、什么情况下保持沉默"。沉默契约是 heartbeat 可用的前提。

**3. 手动验证。** `openclaw cron list` 查看注册的 job，`openclaw cron run <job-id>` 立即触发一次验证投递；heartbeat 则盯 gateway 日志，确认每个 beat 的耗时和是否静默。

**4. 混用分工。** 日报、周报、定时提醒归 cron；收件箱巡逻、TODO 状态检查归 heartbeat。两边输出模板刻意区分：晨报有固定格式，巡逻消息只在有发现时出现，避免重复轰炸。

## 踩坑点

1. **时区。** cron 表达式默认按宿主机时区解析，服务器在 UTC 时"9 点晨报"会变成下午 5 点。显式写 `tz`。
2. **payload 冷启动。** 每次 cron 触发是独立 turn，job 不记得上次说了什么。message 必须自包含，或明确让 agent 先读某个工作区文件。
3. **心跳太密。** 每次 beat 都是一次完整调用，间隔调到 5 分钟后 token 成本线性上涨，且几乎全在静默。先用 30 分钟跑一周再调。
4. **错过的调度不补跑。** 网关停机期间的 cron 不补触发，heartbeat 也跳过错过的 beat。强时序提醒要有兜底，比如关键日程同时挂一个前置 cron。
5. **HEARTBEAT.md 写成时间表。** 心跳只控制频率、不控制时刻，把"每天 9 点汇报"写进心跳文件是无效的，时刻需求必须走 cron。

## 可复用建议

- 选型只问两个问题：**错过一次会怎样？安静时该不该出声？** 前者决定是否需要 cron 的确定性，后者决定 heartbeat 的沉默契约怎么写。
- 新增定时需求先试着合并：一个 heartbeat 能覆盖的巡逻项，不要拆成十个 cron。
- 定期 `openclaw cron list` 清理失效 job，像维护 crontab 一样维护它。

## 总结

cron 是确定性调度器，heartbeat 是带模型判断的巡逻循环，本质是"时刻驱动 vs 事件驱动"。两者不是替代关系：时刻表交给 cron，观察与判断交给 heartbeat。而控制成本的核心始终是——更少、更安静的唤醒。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/fbdefd785b9daf37.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/3ccfedcdd3924b37.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/a4dfc9f631f6a4b7.png)

