---
title: cron 还是 heartbeat？OpenClaw 定时任务的选型与踩坑
feedId: 39783
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

OpenClaw 里让 agent"定时醒来"有两条路：**cron** 和 **heartbeat**。cron 是调度器驱动的精确任务，到点必触发；heartbeat 是周期性心跳，每隔一段时间给 agent 发一条默认 prompt，由 agent 自己决定做不做。很多人上手时把两者混着用，结果要么任务不触发，要么 agent 半夜刷屏。

## 问题

核心是三个：

1. 什么场景该用 cron，什么场景该用 heartbeat？
2. 两者能不能混用，怎么避免重复执行？
3. token 成本和可靠性怎么平衡？

## 两种机制的本质区别

| | cron | heartbeat |
|---|---|---|
| 触发方式 | cron 表达式，调度器决定 | 固定间隔，到点发 prompt |
| 决策者 | 你（prompt 写死） | agent（读 HEARTBEAT.md 自行判断） |
| 时间精度 | 分钟级、确定 | 有间隔漂移，agent 可选择不动 |
| 适合 | 定时日报、定期清理、定点提醒 | 巡检收件箱、环境监控、"有事再做" |

一句话判断：**触发时间是否确定、要不要 agent 自己判断**。时间确定 → cron；只有条件成立才需要动作 → heartbeat。

## 做法

**cron 的正确姿势：**

```bash
openclaw cron add \
  --name "daily-standup" \
  --schedule "0 9 * * 1-5" \
  --message "汇总昨晚的告警和 PR，输出三行摘要" \
  --deliver
```

关键是 prompt 要像下指令：目标、数据来源、输出格式、是否投递，一个都别省。cron 任务跑在隔离 session 里，不带主对话上下文，写模糊了它就自由发挥。

**heartbeat 的正确姿势：**

在 workspace 的 `HEARTBEAT.md` 里写清单，核心是"默认沉默"：

```markdown
## 规则
- 默认不做任何事，直接回复 HEARTBEAT_OK
- 仅当以下条件满足时才行动：
- GitHub 有带 bug 标签的新 issue → 总结并通知我
- 磁盘使用率超过 90% → 告警
```

间隔通过配置里的 `heartbeat.intervalMs` 调整（默认 30 分钟），配合 `activeHours` 限制时段，避免半夜空转。

## 踩坑点

1. **heartbeat 间隔调太小**。有人设 1 分钟想要"实时响应"，结果一天几百次空转，token 账单很诚实。高频监控别用 agent 心跳，用外部脚本 + webhook 触发更省。
2. **HEARTBEAT.md 没写"默认沉默"**。agent 会倾向于"表现一下"，每次心跳都回复点东西，消息渠道被刷屏。
3. **cron 时区**。网关容器里默认 UTC，`0 9 * * *` 实际是北京时间 17 点。要么改容器时区，要么在表达式里显式处理。
4. **cron prompt 太随意**。"帮我看看今天的新闻"这类 prompt 输出不可复现，务必写清来源和格式。
5. **同一件事两边都配了**。cron 每天 9 点发日报，HEARTBEAT.md 里又写"日报相关的事主动做"，结果一天两份日报。定时类任务只挂 cron，heartbeat 清单里删掉。

## 可复用建议

- 判断口诀：**有明确时间点用 cron，有明确条件用 heartbeat，既要精确又怕成本就用外部触发 + webhook。**
- cron 任务全部 prompt 模板化，投递与否显式声明，不依赖默认值。
- heartbeat 条件写成可判定的布尔式，不写"多关注一下"这种模糊语。
- 新任务先低频跑一周，看日志里的触发次数和 token 消耗，再调频率。
- 定期 `openclaw cron list` 审计一遍，删掉没人看的任务。

## 总结

cron 和 heartbeat 不是二选一，而是分工：cron 管"确定性时间"，heartbeat 管"条件性巡检"。把时间敏感的交给调度器，把判断交给 agent，各自做擅长的事，定时任务才稳定可预期。

---

