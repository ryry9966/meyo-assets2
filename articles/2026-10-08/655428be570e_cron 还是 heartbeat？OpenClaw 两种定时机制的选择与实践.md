---
title: cron 还是 heartbeat？OpenClaw 两种定时机制的选择与实践
feedId: 40844
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景

OpenClaw 里有两种让 agent 按时间动起来的机制，很多人第一次用都会混：

- **cron**：经典 cron 语义。每个 job 绑定一条指令和调度表达式（或一次性 `at` 时间），到点唤醒 agent 执行，结果投递到指定会话或频道。任务持久化，重启不丢。
- **heartbeat**：主 agent 的心跳。按固定间隔（默认 30 分钟，可配）把 workspace 里 HEARTBEAT.md 的检查清单注入会话，agent 自己判断这次要不要干活——没事就回 `HEARTBEAT_OK` 跳过，有事才真正执行。

| | cron | heartbeat |
|---|---|---|
| 触发 | 固定时间/表达式 | 固定间隔 |
| 适用 | 时间已知 | 状态未知 |
| 会话 | 可跑隔离会话 | 默认主会话 |
| 成本 | 按次可预估 | 依赖 HEARTBEAT_OK 短路 |

一句话：cron 是"到点派活"，heartbeat 是"定期巡逻"。选错不是不能用，而是有代价——要么错过时间，要么烧 token。

## 怎么选：一条主判断线

问自己：**触发条件是"时间已知"还是"状态未知"？**

- 时间已知（每天 9 点晨报、周五周报、明早 8 点提醒）→ **cron**。调度确定、上下文干净、成本好算。
- 状态未知（"有新邮件就整理"、"仓库出新 PR 就看一眼"、"磁盘超 85% 告警"）→ **heartbeat**。你说不清什么时候触发，就让 agent 每次心跳自己评估。

经验比例：七成"我想要个定时任务"的需求，答案都是 cron。heartbeat 是哨兵，不是通用调度器。

## 做法

**用 cron 做定点任务**：直接下指令让 agent 建，比如"每天 09:00 把日历和未读消息汇总成晨报发到 Telegram"，agent 会生成 `0 9 * * *` 的 job 并配好投递目标。一次性提醒用 `at` 调度，跑完即清。

**用 heartbeat 做状态巡逻**：HEARTBEAT.md 写短清单，例如：

```
- 检查 ~/inbox 有无未处理文件，有则总结并通知
- 磁盘使用率超 85% 才告警
- 都没有：回复 HEARTBEAT_OK
```

最后一行是关键——给 agent 明确的"无事发生"出口，这是成本可控的前提。

**混合用法**：heartbeat 巡逻中发现"这事其实每天固定要做"，让 agent 顺手固化成 cron job；两者是衔接关系，不是二选一。

## 踩坑点

1. **把 heartbeat 当调度器**。间隔是近似值，会话忙时心跳可能被跳过或排队，指望它精确触发会失望。精确时间交给 cron。
2. **成本失控**。30 分钟一次 × 48 次/天 × 主会话上下文，一天不少 token。对策：拉长间隔、HEARTBEAT.md 控制在十行以内、坚持 HEARTBEAT_OK 短路。
3. **污染主会话**。heartbeat 默认跑在主会话，每次都往上下文里塞东西。把它指到独立会话，主会话只收结论。
4. **cron 静默失败**。job 跑了但没收到，多半是 delivery 没配或目标会话不存在。建完先手动触发一次验证链路。
5. **时区**。cron 表达式按 gateway 主机时区解释，容器里常见 UTC。晨报变午报基本都是这个原因。

## 可复用建议

- heartbeat 检查项务必带**阈值和去重**（"超 85% 才告警"、"同一问题 24h 内不重复报"），否则巡逻会变骚扰。
- cron 的指令写成自包含的，不依赖既有上下文；确实需要上下文就明确让它跑主会话。
- 观察实际运行一周再调参：间隔、清单精简度、投递路径，都拿真实日志说话。

## 总结

cron 是日程表，heartbeat 是值班室。时间确定用 cron，状态未知用 heartbeat，再用"heartbeat 发现规律 → 固化成 cron"衔接两者。先问触发条件是时间还是状态，答案通常就出来了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/dbeb05e2f79067fa.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/8607e496363c8df5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/12baf3ba509b4758.png)

