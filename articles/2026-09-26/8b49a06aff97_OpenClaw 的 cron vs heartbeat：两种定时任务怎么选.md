---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 39157
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

OpenClaw 网关是常驻进程，跑在自己的机器上，很多人拿它做自动化：早报摘要、定期巡检、到点提醒。OpenClaw 原生提供了两条定时路径：**heartbeat（心跳）** 和 **cron（排程任务）**。两者都能让 agent"自动动起来"，但设计意图完全不同，混着用是 token 成本失控和消息噪音的主要来源。

## 机制差异

**heartbeat 是节拍驱动。** 全局只有一个心跳循环（默认约 30 分钟一拍，`heartbeat.every` 可调）。每一拍，网关给 agent 发一个轻量唤醒 prompt，agent 读取 workspace 里的 `HEARTBEAT.md`，按清单检查一遍，没事就回 `HEARTBEAT_OK` 静默收场，有事才向目标会话/渠道发消息。关键点：**每一拍都是一次真实的模型调用**，无论有没有事。

**cron 是时刻驱动。** 每个 job 显式声明表达式、时区、payload、目标 agent 和投递渠道，到点精确触发一次，跑完即止，且可以跑在 isolated session 里，不污染主对话上下文。

一句话：heartbeat 回答"这一拍有没有要看的？"，cron 回答"X 时 X 分该做什么"。

## 怎么选

- 固定时刻、固定动作、明确投递对象 → **cron**。例如工作日 9 点发日报、每晚 23 点整理收藏。
- "每隔一段时间看一眼，视情况处理" → **heartbeat**。例如检查后台任务是否跑完、有没有需要介入的异常。
- 实际项目里我最终都是混合：**cron 干重活，heartbeat 只做轻量巡检**。

## 配置步骤（示意，参数以 `--help` 为准）

**heartbeat：**
1. 在 `~/.openclaw/workspace/HEARTBEAT.md` 写清单，每行一个检查项，附明确的触发条件和"否则静默"规则。
2. 配置 `heartbeat.every` 与投递目标，先用 30–60 分钟起步。
3. 跑一周，看日志里的心跳次数和 token 用量，再决定收紧或放宽间隔。

**cron：**
1. `openclaw cron add --name "daily-report" --cron "0 9 * * 1-5" --tz Asia/Shanghai --session isolated --message "汇总昨日会话要点并投递"`（示意）。
2. `openclaw cron list` 确认注册，手动触发一次验证投递链路。
3. 一次性任务跑完记得清理，避免残留 job 到点空转。

## 踩坑点

- **心跳间隔调太短是最大的成本坑。** 每 5 分钟一拍就是每天 288 次模型调用，哪怕全是 HEARTBEAT_OK。
- **重活别放 heartbeat。** 心跳会拼上主会话上下文，任务一重、对话一长，单拍成本立刻上去。
- **cron 的时区坑。** 服务器多半是 UTC，不显式指定 `--tz`，"早上 9 点"会变成下午 5 点。
- **cron job 写进共享 session** 会把执行痕迹留在主对话里，几次之后上下文明显变脏；一律用 isolated。
- **HEARTBEAT.md 写太宽泛**（比如"检查一切异常"）会导致每拍都产出噪音消息；必须写清"什么算有事，什么叫没事"。
- **补跑语义不同。** cron 错过触发点就是错过；heartbeat 天然是"下一拍自然会来"。过期也无妨的任务适合心跳，必须准时的任务只能用 cron。

## 可复用建议

- 心跳管**状态**，cron 管**事件**。
- 把 HEARTBEAT.md 当 SLA 清单维护：每项 = 检查对象 + 触发条件 + 静默规则。
- 到点必达的任务：cron + isolated session + 显式投递渠道，三件套配齐。
- 每月看一次心跳调用次数，成本涨了先怀疑间隔设置和清单膨胀。

## 总结

两者不是竞争关系，而是分工：cron 负责"准时、精确、可审计"的任务，heartbeat 负责"低频、模糊、依赖当下上下文"的巡检。想清楚你要的是**时刻**还是**节拍**，选型基本不会错。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/9adb56bc2e9513ff.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/576b3e683db8e84d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/4a97ae3adb84d328.png)

