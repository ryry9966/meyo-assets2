---
title: OpenClaw 定时任务选型：cron 与 heartbeat 的分工边界
feedId: 41960
source: 综合讨论
publishedAt: 2026-10-19
---

## 背景

OpenClaw 里有两套"让 agent 定时干活"的机制，新人很容易混着用：

- **cron**：标准 cron 表达式驱动，到点触发一次 agent，执行你预设的 payload。
- **heartbeat**：agent 按固定间隔（默认 30 分钟）自己醒来，读一遍工作区的 `HEARTBEAT.md`，自行判断这轮有没有事可做，没事就回 `HEARTBEAT_OK` 继续睡。

两者触发语义不同：cron 是"到点叫它干指定的事"，heartbeat 是"按时自查有没有事"。

## 问题

我最初的用法很典型：所有周期性需求全塞进 `HEARTBEAT.md`。结果：

1. token 消耗线性上涨——每次醒来都读完整清单，而 90% 的条目这轮根本轮不到；
2. "每天早上 9 点发日报"这类有明确时刻的任务，触发时间持续漂移，heartbeat 的间隔是"至少这么久"，不是"准点"；
3. 反面案例也有：把"留意某服务异常"拆成 5 个每 10 分钟的 cron 任务，列表臃肿还互相踩。

核心矛盾在于：**两种机制语义不同，混用会同时付出精确性和成本的代价。**

## 做法：一条判断主线，三步落地

判断主线一句话：

> **有确定时刻 → cron；只有不确定条件 → heartbeat。**

1. **按"时刻 vs 条件"分桶。**"每周一 9 点整理 issue"是时刻，进 cron；"盯着我部署的服务，异常再喊我"是条件，写进 `HEARTBEAT.md`。
2. **cron 任务选对会话。**cron 默认在隔离会话里跑，agent 没有主会话记忆。要它带着上下文做事，用 targeted 投递到主会话；只要产出报告就保持隔离，避免污染主上下文。
3. **heartbeat 做减法。**`HEARTBEAT.md` 只留"持续关注"类短条目，一行一条；没有需要轮询的事就清空或拉长间隔，别让它空转。顺手用 `cron list` 定期审任务列表，删掉一次性的。

## 踩坑点

- **heartbeat 间隔太短。**"5 分钟查一次"看着勤快，实际每 5 分钟烧一轮 token。先评估真实时效要求，多数巡检场景 30 分钟够用。
- **`HEARTBEAT.md` 变垃圾抽屉。**临时需求做完忘删，之后每次醒来都为它付费，清单必须定期清。
- **cron 时区。**宿主机时区和你想表达的时区不一致时，日报会在奇怪的钟点发出来，建任务时显式确认。
- **同一任务双写。**cron 里建了"每天同步"，`HEARTBEAT.md` 里又写"记得同步"，会重复执行。一个任务只归一个机制管。
- **长任务重叠。**heartbeat 上一轮没跑完下一轮又醒，主会话里任务穿插。重活挪到 cron 隔离会话跑。

## 可复用建议

- 把 heartbeat 理解为"巡检"，cron 理解为"闹钟"，语义对了选型自然对。
- 监控类需求优先合并成一条 heartbeat 条目，而不是 N 个高频 cron。
- 高质量定时产出（日报、周报）的 payload 里写清范围、格式和"不做的事"。
- 每月做一次调度审计：`cron list` 过一遍 + `HEARTBEAT.md` 清一遍，成本和信噪比都会明显改善。

## 总结

cron 和 heartbeat 不是竞争关系，是分工关系：**时刻驱动交给 cron，条件驱动交给 heartbeat，并保证一个任务只有一个归属**。守住这条边界，OpenClaw 的定时自动化就能同时做到准时、省钱、少打扰。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/40e331089a4bd35a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/c69a58127e1c2de2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/7944760c520d95c2.png)

