---
title: cron 还是 heartbeat？OpenClaw 定时任务的选型笔记
feedId: 41064
source: 综合讨论
publishedAt: 2026-10-10
---

## 背景

在 OpenClaw 里让 agent"定时干活"，有两条路：**cron**——由 gateway 的 cron 模块调度的精确计划任务；**heartbeat**——周期性心跳，agent 每隔 N 分钟被唤醒一次，自己判断要不要做事。两者都能做自动化，但调度模型完全不同，选错会出现很典型的症状。

## 问题

我在实际使用中踩过的几个坑，基本都源于选型错误：

- 想要"每天 8:30 发日报"，却用了 heartbeat，结果 8:07 或 8:53 才来，时间不可控；
- 想要"有值得提醒的事再叫我"，却用了 cron，每小时白跑一次，没必要时也在打扰；
- 心跳间隔调到 5 分钟，token 消耗翻了几倍；
- cron 任务跑完了，结果没人收到，排查半天发现是投递没配。

## 两种机制的本质区别

- **cron：时间驱动**。到点就执行，默认跑在隔离会话（isolated session）里，可以显式指定投递渠道。适合"确定时刻 + 确定动作"。
- **heartbeat：状态驱动**。agent 周期性醒来，读工作区里的 `HEARTBEAT.md`，自己判断有没有值得做的事，没有就静默（返回 `HEARTBEAT_OK`）。适合条件触发的主动性任务。它跑在主会话里，天然带对话上下文。

## 怎么选：三问

1. 触发条件是**时刻**还是**状态**？"周一 9 点"→ cron；"如果……就告诉我"→ heartbeat。
2. 需要对话上下文吗？需要→ heartbeat，或把上下文写进 cron 的 payload。
3. 结果要精确投递到某个渠道吗？要→ cron 配 delivery。

## 做法

**cron 示例**（具体 flag 以 `openclaw cron --help` 为准，版本间略有差异）：

```bash
openclaw cron add \
  --name morning-digest \
  --cron "30 8 * * *" \
  --tz Asia/Shanghai \
  --session isolated \
  --message "汇总我昨晚的未读消息与今日日历，生成 10 行以内的摘要" \
  --deliver
```

**heartbeat 示例**：在 workspace 写 `HEARTBEAT.md`，把它当成一份 SLA 来写：

```markdown
# Heartbeat
- 每次心跳检查 ~/inbox/today.md 有无今日待办；没有则输出 HEARTBEAT_OK
- 只有出现"今天就过期"的条目才主动通知我
- 禁止在心跳里做抓取、生成等重活；发现重活需求，记入 ~/inbox/task.md
```

心跳间隔在 `openclaw.json` 的 `agents.defaults.heartbeat` 下调整，建议先保持默认观察一周再动。

## 踩坑点

1. `HEARTBEAT.md` 没写"无事返回 HEARTBEAT_OK"，agent 每次心跳都发消息，人被轰炸；
2. 心跳里干重活：长任务阻塞主会话、污染上下文，甚至和你聊天时"卡住"，重活一律走 cron isolated；
3. cron 时区：gateway 服务器常是 UTC，不显式指定 `--tz`，早上 8:30 变成下午 4:30；
4. cron 没配投递，或用了 next-heartbeat 唤醒模式，结果延迟到下个心跳才执行；
5. 一次性任务赶上 gateway 重启会错过，重要任务改用时间窗口判断或加补偿逻辑；
6. 心跳间隔过短 + 大模型 = 隐性成本，先算清"次数 × 单次 token"再调频。

## 可复用建议

- 一句话选型：**日历型需求用 cron，条件型需求用 heartbeat**；混合需求用 heartbeat 做哨兵、cron 做执行；
- cron 任务写成幂等的：重跑不产生重复副作用；
- `HEARTBEAT.md` 纳入版本控制，改一句话就能调整 agent 的"值班性格"；
- 定期 `openclaw cron list` 审计任务，删掉不再需要的，别让僵尸任务烧 token。

## 总结

cron 是闹钟，heartbeat 是值班。闹钟保证准时，值班保证该出手时才出手。多数自动化不是二选一：用 heartbeat 做轻量巡检，用 cron 做确定性执行，各司其职，成本和行为才都可控。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/035a39c3182e502a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/67073f4626552a25.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/d5bd09132aa58b4d.png)

