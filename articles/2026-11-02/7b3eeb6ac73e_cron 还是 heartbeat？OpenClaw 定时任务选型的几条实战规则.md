---
title: cron 还是 heartbeat？OpenClaw 定时任务选型的几条实战规则
feedId: 43240
source: 综合讨论
publishedAt: 2026-11-02
---

## 背景

OpenClaw 里能让 agent"自己动起来"的机制有两种：cron（定时任务）和 heartbeat（心跳）。两者都能周期性触发 agent，但设计目标完全不同。新手上手最容易犯的错，就是把所有自动化都塞进其中一种，直到 token 烧了或者时间点永远对不上才发现问题。

## 问题

典型场景一：每天早上 9 点把昨日未读消息汇总发到 Telegram——用 heartbeat 做，时间不可控，还要占用主会话。
典型场景二：随时检查有没有需要跟进的事项（收件箱分诊、待办提醒）——用 cron 做，得为每个事项单独建 job，而且拿不到主会话的上下文。

先说结论：**cron 管"几点做"，heartbeat 管"要不要做"。**

## 两者到底差在哪

- **heartbeat**：每个 agent 一条持久心跳循环（默认 30 分钟，`agents.defaults.heartbeat.every` 可调）。每次醒来读取 workspace 里的 `HEARTBEAT.md`，逐条判断"现在该不该做"，没事就静默跳过。它跑在主会话里，能看到完整对话历史和记忆。
- **cron**：独立调度器，标准 5 段 cron 表达式（也支持 `every` 周期和一次性 `at`）。每个 job 默认隔离会话，payload 精确注入，跑完可以把结果投递到指定渠道。任务持久化在 `~/.openclaw/cron/jobs.json`，重启不丢。

## 做法与步骤

1. **精确时间的任务用 cron**：

```bash
openclaw cron add \
  --name "daily-digest" \
  --cron "0 9 * * *" \
  --session isolated \
  --message "汇总昨日未读消息并生成摘要" \
  --deliver --channel telegram --to "<chat-id>"
```

先用 `openclaw cron run daily-digest` 手动跑一遍，确认输出和投递都正常，再让它按计划走。跑完可以用 `openclaw cron list` 查最近一次执行状态。

2. **条件触发的轻量任务放进 `HEARTBEAT.md`**，每行一条，保持清单短小。心跳间隔不建议低于 15 分钟——每次醒来即使什么都不做，也是一次真实的模型调用。

3. **需要共享主会话记忆的定时任务**，cron 加 `--session main`（或指定 session key），但要想清楚这会持续污染主会话上下文。

## 踩坑点

- **时区**：容器里默认 UTC，`0 9 * * *` 实际是北京时间 17 点。给 gateway 进程显式设 `TZ=Asia/Shanghai`。
- **heartbeat 不对齐整点**：它是"自上次运行 + 间隔"，gateway 重启后重新计时，别拿它做准点提醒。
- **隔离会话没记忆**：cron 的隔离 job 看不到主会话历史，需要背景信息就写进 prompt，或明确改用共享会话。
- **投递失败不可见**：announce 的渠道没连上时，结果只留在 session 日志里。第一次配置务必手动验证一次投递链路。
- **集中触发**：一堆 job 都配在整点，容易撞模型限流，把分钟位错开。
- **清单膨胀**：`HEARTBEAT.md` 塞了十几条之后，每次心跳的评估成本和上下文都会明显上涨，做完一条删一条。

## 可复用建议

- 决策口诀：**固定时间 → cron；条件判断 → heartbeat；要带记忆 → cron + 共享会话（慎用）**。
- 报告类、摘要类任务一律 cron + 隔离会话 + 显式投递，输出与主会话彻底解耦，失败也好排查。
- `HEARTBEAT.md` 控制在个位数条目，把它当便签而不是任务系统；重活都迁去 cron。
- 心跳配 `activeHours` 限制在白天活跃时段，省 token，也避免半夜被 agent 吵醒。

## 总结

cron 是可靠的闹钟，heartbeat 是廉价的巡逻。绝大多数"定时"需求应该落在 cron 上；heartbeat 留给那种"说不上几点，但发生了就该处理"的杂务。两条机制各司其职，agent 才能既准时，又不悄悄烧掉你的配额。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/5d90d6018ca89b70.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/50cb11e763590bd1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/a28e2de63a999ce9.png)

