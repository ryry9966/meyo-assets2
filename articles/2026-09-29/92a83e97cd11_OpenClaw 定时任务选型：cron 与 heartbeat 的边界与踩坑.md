---
title: OpenClaw 定时任务选型：cron 与 heartbeat 的边界与踩坑
feedId: 39585
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

用 OpenClaw 搭自动化，几乎人人都会在同一个路口犹豫：一件事需要"定时"发生，到底是加一条 cron 任务，还是丢进 `HEARTBEAT.md` 让心跳去管？两套机制并存，文档各讲各的，边界往往要自己踩过才知道。

## 问题

我早期犯过的错误：把"每天 8 点发日报"写进 `HEARTBEAT.md`，指望 agent 到点自觉执行。结果心跳间隔是浮动的、agent 还握有判断权，日报经常 8:40 才出来，偶尔干脆被跳过。反过来，也有人把十几条巡检逻辑全堆成 cron 表达式，每加一条检查就要维护一个 job，改一次逻辑动五六个文件。

本质区别一句话：**cron 是日历驱动**，时间到就执行，确定性强；**heartbeat 是状态驱动**，按节奏醒来，看一眼有没有事，有事才做。

## 做法

选型前问自己三个问题：

1. 有没有必须命中的时间点？发日报、定时备份、整点拉数据——用 cron。
2. 还是只是定期看一眼、有条件才动作？目录变化巡检、MCP 服务健康检查、新文件摘要——用 heartbeat。
3. 任务需要主会话的上下文吗？cron 默认隔离 session，不共享聊天记忆；heartbeat 贴着常驻上下文走。

cron 侧的原则是"输入自带、输出可观测"：

```bash
# 示意，参数以你的版本为准
openclaw cron add --name daily-report \
  --cron "0 8 * * *" \
  --message "汇总昨日 workspace 变更并生成日报" \
  --deliver telegram
```

payload 里把输入写全，别指望它记得上次聊了什么；结果投递到 channel，失败才有感知。

heartbeat 侧把 `HEARTBEAT.md` 当成唯一巡检入口：每条写清触发条件，并明确"无事可做时直接回复 HEARTBEAT_OK"尽早短路。间隔按需调，默认 30 分钟对多数巡检够用，想压到 5 分钟之前先算一笔 token 账。

## 踩坑点

1. **时区**：容器里默认可能是 UTC，cron 整体提前 8 小时。出过一次之后，我给所有 schedule 都加了时区校验。
2. **heartbeat 不是闹钟**：间隔语义是"至少这么久"，忙时会顺延，有 deadline 的事别交给它。
3. **cron 的重入与幂等**：上一轮没跑完、或 agent 失败重试时任务会再执行一遍，写文件、发通知这类动作要设计成幂等。
4. **心跳空转**：`HEARTBEAT.md` 写得太含糊（比如"看看有什么要做的"），agent 每轮都会认真思考一圈，token 直接翻倍。清单越具体，短路越快。

## 可复用建议

- 口诀：**有确定的时刻用 cron，只有确定的节奏用 heartbeat**。
- 组合用法：heartbeat 当哨兵发现条件、触发一次性动作；cron 承接所有日历型任务。别让一个机制去干另一个的活。
- cron 任务输入自包含，输出落到 channel 或文件，方便事后排障。
- token 账单异常时，先拧 heartbeat 间隔这个最直接的成本旋钮。
- 排查"任务没跑"先分两类：没触发（schedule/时区问题），还是触发了没做完（查 session 日志）。

## 总结

cron 与 heartbeat 不是二选一，而是分工：cron 管"什么时候必须做"，heartbeat 管"空闲时值得看什么"。把边界划清，OpenClaw 的自动化才不会变成一笔糊涂账。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/2e7e86efd999fd20.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/c44f6bea49a92d68.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/bf2b6932c8f9162b.png)

