---
title: cron 还是 heartbeat？OpenClaw 两种定时机制的分工与踩坑
feedId: 37447
source: 综合讨论
publishedAt: 2026-09-14
---

用 OpenClaw 跑了一段时间后，几乎每个人都会遇到同一个问题：想让 agent "定时干活"，到底是加一条 cron job，还是写进 HEARTBEAT.md 让 heartbeat 自己巡检？这篇把我们踩过的坑和最终的分工原则整理一下。

## 背景：两套机制，解决的是两类问题

OpenClaw 网关里这两种调度看起来都是"周期性唤醒 agent"，但设计目标不同：

- **cron job**：精确的定时器。到点触发一次完整的 agent turn，跑完即止。适合"每天 8 点发日报"这种时刻驱动任务。
- **heartbeat**：巡检循环。agent 每个周期（默认约 30 分钟，可调）在主会话里醒来一次，读一遍 HEARTBEAT.md，自己判断有没有事要做，没有就保持沉默。适合"有情况才动作"的条件驱动任务。

最关键的区别只有一条：**heartbeat 的判断发生在主会话里，能用上最新的对话上下文；cron 默认跑在 isolated 会话里，是一份无状态快照。**

## 做法

### cron：当闹钟用

```bash
openclaw cron add --name "morning-brief" \
  --cron "0 8 * * *" \
  --session isolated \
  --message "汇总昨天的群消息，生成 10 条以内的摘要" \
  --deliver --channel telegram --to "<chat_id>"
```

三个要点：
1. isolated 会话不共享主会话历史，prompt 必须自包含——需要的事实直接写进 message，或让它去读工作区文件。
2. 不加 `--deliver`，结果只留在会话里，不会推送到渠道。
3. 用 `openclaw cron list` 和 `openclaw cron runs` 核对执行记录，上线后先盯两天。

### heartbeat：当哨兵用

在 `~/.openclaw/workspace/HEARTBEAT.md` 里写条件式清单，例如：

```
- 若有 @ 我且超过 2 小时未处理的消息，提醒我并给出建议回复
- 若磁盘剩余低于 20%，发一条告警
- 其余情况：回复 HEARTBEAT OK
```

原则是**只写条件和动作，不写任务描述**。判断权交给 agent，这才有"巡检"的价值。

## 踩坑点

1. **heartbeat 不是准点时钟**。机器睡眠时暂停、错过不补、触发时间有漂移。任何"必须准点"的需求都别放这里。
2. **每次 heartbeat 醒来都是一次真实推理**。HEARTBEAT.md 写得越长，空转成本越高。把日报这种活儿塞进 heartbeat，等于买了一张全天候烧 token 的年票。
3. **cron 的 isolated 会话没有记忆**。别指望它记得昨天说了什么；需要连续性就让它读文件，或把关键上下文写进 message。
4. **时区**。cron 表达式按网关本地时间解析，容器里 UTC 偏 8 小时是经典事故，部署后先 `date` 验证。
5. **离线期间的 cron 视版本可能不补跑**，重要任务让 agent 自己维护一个"上次成功时间"用于对账。

## 可复用的选择规则

- 时刻驱动 → cron；条件驱动 → heartbeat。
- 周期性产出（日报、周报、定时提醒）= cron + isolated + deliver。
- 监控、等待、依赖最新对话状态的判断 = heartbeat。
- HEARTBEAT.md 控制在 5 行以内，越短越便宜也越可靠。
- 同一个任务不要两边都挂；确实要联动，就让 cron 触发的 turn 自己决定是否唤醒 heartbeat。
- 成本敏感时先拉长 heartbeat 间隔，观察日志里的空转率再收紧。

## 总结

cron 和 heartbeat 不是替代关系，是分工关系：一个是到点就响的闹钟，一个是常开的哨兵。选型时不要问"哪个更像定时任务"，而是问两个问题：**触发依据是时刻还是条件？执行时需不需要主会话上下文？**想清楚这两问，配置基本不会错。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/c712658bf54f1df3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/106b53fb65598ef6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/aab0740505f75112.png)

