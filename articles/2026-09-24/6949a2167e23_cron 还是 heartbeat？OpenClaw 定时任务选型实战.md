---
title: cron 还是 heartbeat？OpenClaw 定时任务选型实战
feedId: 38816
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

OpenClaw 里有两套让 agent「定时醒」的机制，新手很容易混淆：

- **cron**：传统意义的定时触发。注册一条任务，写清楚时间表达式和 prompt，到点 agent 被唤醒一次，执行、推送结果、结束。触发即任务，做完即停。
- **heartbeat**：周期性自唤醒。agent 每隔一段时间（默认约 30 分钟，可调）自己醒一次，读 `HEARTBEAT.md` 里的巡检清单，结合上下文判断「现在有没有值得做的事」。有事就做，没事就静默结束，不产生输出。

两者解决的问题有重叠但不相同，混着用很容易出事。

## 问题

几个我实际踩过的场景：

1. 把「每天 9 点发日报」写进了 heartbeat 清单，结果 agent 在 8:47 那次唤醒就把日报发了，时间完全不可控。
2. heartbeat 清单里写了句「看看有没有要跟进的事」，agent 每次唤醒都认真思考一遍，token 消耗翻倍，但大多数时候本来什么都不该做。
3. 同一个监控项既配了 cron 又出现在 heartbeat 里，用户收到重复通知。

本质区别一句话：**cron 是时间驱动，heartbeat 是状态驱动**。

## 做法

先给任务分类，再选机制。

**用 cron 的场景**：执行时间本身就是需求，日报、定时提醒、每天固定时间抓数据。配置要点：

- 时间表达式写精确，确认运行环境的 `TZ`，容器里时区漂移是常客；
- prompt 写死输出格式和投递目标，不留发挥空间；
- 设超时，避免任务挂死占着会话。

**用 heartbeat 的场景**：关心「状态是否变化」而不是「几点执行」——盯收件箱、检查服务健康、跟进未完成任务。配置要点：

- `HEARTBEAT.md` 当 runbook 写：每条一句话，包含明确触发条件和动作，例如「若仓库有未处理的 issue 评论，汇总后通知；否则保持静默」；
- 显式写**静默条件**，这是省 token 的关键；
- 间隔先调大做灰度：本地 30 分钟起步，确认没有空转再收紧。

**组合策略**：heartbeat 只负责发现和上报；确定性的重复动作（整理、汇总、发送）仍交给 cron。heartbeat 发现异常后写入任务列表，由 cron 到点处理，链路最清晰。

## 踩坑点

- **HEARTBEAT.md 越写越长**：每次唤醒都会注入上下文，成本线性上涨，要定期删掉失效条目。
- **heartbeat 静默失败无感知**：一次唤醒挂了什么都不会发生，重要巡检要有「超过 N 小时无心跳日志就告警」的兜底。
- **cron 的 prompt 太开放**：agent 会自由发挥，输出格式漂移，给它模板。
- **重复通知**：一个监控项只允许归属一个机制，建议在清单头部备注归属关系。

## 可复用建议

选型口诀：**到点必须做的事给 cron，有空看一眼的事给 heartbeat；cron 管输出，heartbeat 管状态。**

另外两条经验：

1. 把 `HEARTBEAT.md` 当代码 review，每次改动留一句说明，避免清单腐化；
2. 上线前先跑一周「只记录不执行」模式，观察唤醒次数与实际动作的比例，再决定间隔是否收紧。

## 总结

cron 和 heartbeat 不是替代关系。时间敏感的任务交给 cron，状态敏感的巡检交给 heartbeat，并保证每件事只有一个归属——这样 OpenClaw 的自动化才既省 token，又可靠。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/efcdd4944bee529c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/1cd411f16777ed37.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/09c91fc9ea61a886.png)

