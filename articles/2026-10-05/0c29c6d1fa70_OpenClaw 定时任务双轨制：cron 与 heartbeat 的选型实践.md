---
title: OpenClaw 定时任务双轨制：cron 与 heartbeat 的选型实践
feedId: 40550
source: 综合讨论
publishedAt: 2026-10-05
---

## 背景

OpenClaw 的 agent 常驻运行后，"让 agent 定时干活"有两条路：

- **cron**：网关内置的作业调度器，按时间表达式触发，把一条 message 投进指定 session，并可把执行结果投递到聊天渠道。通过 `openclaw cron list / add / edit / rm` 管理，或直接写进 `openclaw.json` 的 `cron.jobs`。
- **heartbeat**：主 session 的周期心跳，每隔一段时间（默认 30m）注入一条心跳消息，由 agent 自己判断——没事就回 `HEARTBEAT_OK` 保持沉默，有事才真正输出。

两者都挂在 gateway 进程下、走同一个模型通道，但设计意图完全不同。很多翻车都源于把二者混用。

## 问题

新手常见两类极端：

1. **全塞进 heartbeat**：把晨报、提醒、巡检全写进心跳 prompt / HEARTBEAT.md。结果是每个 tick 都全量跑一遍清单，token 消耗大、触发时间随间隔漂移，任务一多互相干扰。社区现在的共识也是：心跳保持轻量，周期任务迁去 cron。
2. **全用 cron**：cron 的 message 是静态的。一旦"要不要做事"取决于当前状态（收件箱有没有新邮件、目录有没有变更），静态提示只能靠模型每次重猜，容易重复执行或漏判。

## 做法

**时间锚点明确的任务交给 cron**。示例配置：

```json
{
  "name": "morning-brief",
  "schedule": { "kind": "cron", "expr": "0 7 * * *", "tz": "Asia/Shanghai" },
  "payload": { "kind": "agentTurn", "message": "汇总昨晚的 GitHub 通知和日历，生成晨报" },
  "session": { "target": "isolated" },
  "delivery": { "mode": "announce", "channel": "telegram", "to": "me" }
}
```

要点：显式指定 `tz`；用 `delivery` 把结果推到渠道，否则跑完没人知道；一次性任务走隔离 session，需要上下文连续的再用固定 target，避免污染主会话。

**状态驱动的低频巡检留给 heartbeat**：

```json
"agents": { "defaults": { "heartbeat": { "every": "45m" } } }
```

心跳 prompt 里必须写清沉默协议：默认回 `HEARTBEAT_OK` 不打扰用户，只有命中明确条件（未读消息、报告目录出现新文件）才输出。心跳只做"读状态 + 判断"，不做重活——重活让心跳决定后转成一次性动作或提醒你去建 cron。

**一条选型规则**：固定时间点/固定间隔必须执行 → cron；"每隔一段看一眼，有事再说" → heartbeat；状态检查但对时点敏感（每小时准点）→ 降级为 cron，在 message 里写清检查步骤。

## 踩坑点

- **时区**：cron 默认用网关所在时区，VPS 上多半是 UTC，晨报会"准点"跑到下午三点。永远显式写 `tz`。
- **心跳噪音**：不写 `HEARTBEAT_OK` 规则，agent 每个 tick 都向渠道发消息，用户很快就会把心跳关掉。
- **心跳过密**：`every: "5m"` 纯属烧钱，对"没有变化的状态"，更密的巡检并不会更早发现什么。
- **cron 挤占主会话**：target 指向 main 的长任务会和你正在进行的对话抢上下文，重活建议隔离 session。
- **漏跑语义**：两者都依赖 gateway 进程存活，休眠/重启期间一律不补跑。检查逻辑要幂等，别假设"上次一定执行过"。

## 可复用建议

- **分层**：heartbeat 只留活性确认 + 轻量状态扫描，一切带时间语义的任务沉淀为 cron。新增任务改配置即可，不用反复改 prompt。
- 心跳频率 ≥ 30m，任务稳定后逐步拉长。
- 定期 `openclaw cron list` 审计，删掉跑了几周没人读结果的任务。
- 每个 cron 任务起可检索的名字，排障时 grep 日志效率完全不同。

## 总结

cron 是"到点必须做"，heartbeat 是"到点看一眼"。前者管确定性，后者管主动性。把确定性从心跳里抽走、把主动性约束进沉默协议，两套机制各司其职，agent 才能做到既准时、又安静。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/4df93f7499a6af86.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/cdbc7632462f9f58.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/1c499a8e8202ce23.png)

