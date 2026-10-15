---
title: AI 助手的 heartbeat 设计：轮询 vs 推送的取舍
feedId: 41610
source: 综合讨论
publishedAt: 2026-10-15
---

## 背景

OpenClaw 这类助手不是一问一答的一次性会话，而是一个常驻 gateway 进程挂着若干会话。要让 agent “主动干活”，最朴素的方案就是 heartbeat：每隔一段时间唤醒一次，让它检查待办、跑巡检，或者什么都不做。机制本身很简单，但“这个定时器放在谁手里”，决定了后面所有的工程细节。

## 问题

两种模式各有归属：

- **push**：定时器长在 gateway 里，到点触发一次 agent run，把结果推到目标会话。
- **polling**：每个客户端自己定时来问。

实际跑起来会遇到三个痛点：

1. 多端轮询会重复唤醒 agent——手机端、桌面端各烧一份 token，还可能各自触发一次主动消息，用户收到两条一样的。
2. push 模式下 gateway 挂了就是静默失败，你以为它在巡检，其实三天没跑。
3. heartbeat prompt 写重了，每 30 分钟一次比正经对话还费 token。

## 做法

我们的取舍原则：**默认 push，除非有明确理由**。

1. **定时器收口到 gateway**。保持默认的 push 心跳，间隔从 30 分钟起步，别一上来就 5 分钟。
2. **心跳 prompt 保持极小**：只写“要检查什么 + 没事就返回 `HEARTBEAT_OK`"。agent 返回 `HEARTBEAT_OK` 时不下发任何消息，这是控制噪音的关键开关。
3. **确定性排程走 cron**：每天早报这类要精确时刻的任务交给 cron，heartbeat 只做模糊巡检，两者别混。
4. **确实需要 polling 时**（比如多端各自要节奏），必须加两样东西：随机抖动（interval ±10%，避免多端同步打点）和一把本地锁/锁文件，保证同一时刻只有一次 run。
5. **每次心跳落一条日志**：时间、决策（ok / acted）、耗时、token 数。用文件或 MCP 状态存一个 `last_beat` 时间戳。

## 踩坑点

- **agent 没按约定返回 `HEARTBEAT_OK`**，频道被“无事发生”刷屏。修法：prompt 里给明确的输出约定，gateway 侧再做一层过滤兜底。
- **间隔太短 + 心跳里调了重工具**，两次 run 重叠排队，表现为“助手越来越慢”。先测单次 run 时长再定间隔，别拍脑袋。
- **休眠/断网后的追赶逻辑**：笔记本睡了两小时，醒来别把错过的 beat 全补跑，只补最近一次或直接跳过。
- **push 的静默失败**：加一个外部 dead-man switch，比如 uptime 监控定时打一个 HTTP 心跳端点，连续几小时没收到就告警。
- **轮询端不幂等**：主动消息重复发。把“这条待办是否已处理”落盘，别放在内存里。

## 可复用建议

- push 是默认解：单一调度点、省 token、好观测；polling 只在“多端各自需要心跳”或“网关不可靠”时才用。
- 心跳总成本 = 间隔 × 单次成本。先把单次成本压小（小 prompt、轻工具、能早返回就早返回），再考虑缩间隔。
- 一切心跳动作必须幂等且可追溯，`last_beat` 时间戳是排障第一入口。
- 心跳预算要单独看：如果日志里心跳 token 占了日常用量的大头，说明该减频或瘦身了。

## 总结

heartbeat 的本质不是技术选型，是**预算和归属**问题：定时器归 gateway（push）还是归客户端（polling），决定了重复执行、静默失败和 token 成本各自出现在哪。从 push + 30 分钟 + 极小 prompt 起步，把观测和幂等做扎实，再谈调频调优。心跳是给助手装“脉搏”，不是给它加“闹钟轰炸”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/81c1c7cb9c776f12.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/0e134cc704aca727.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/e89e4af2bd10c8df.png)

