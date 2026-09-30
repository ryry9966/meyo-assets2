---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 39939
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

跑了一段时间 OpenClaw，几乎人人都会想让 agent「主动干活」：早上推一条日报、定点抓一次数据、每小时巡检服务器。OpenClaw 里能做这件事的入口有两个：**cron（定时任务）** 和 **heartbeat（心跳）**。混用的结果通常是：要么 token 烧得莫名其妙，要么该准点触发的事没准点。这篇按实际使用经验，把两者的边界讲清楚。

## 机制差别

- **cron**：你写死时间表（cron 表达式或一次性任务），到点由网关把一段确定的 payload 交给 agent 执行。触发是确定性的，与模型判断无关。支持时区、jitter、隔离会话、结果投递到频道。
- **heartbeat**：按固定间隔（默认 30 分钟）向主会话注入一次心跳 prompt，agent 读工作区的 HEARTBEAT.md 清单，自己判断「有没有事要做」，没事就回 HEARTBEAT_OK，消息被吞掉不打扰你。

一句话：**cron 是「我决定什么时候做」，heartbeat 是「你定期自己看看要不要做」。**

## 怎么选

决策规则很简单：

- 有明确时刻或节奏、要确定性结果 → **cron**。日报、定点备份、整点拉数据、每周清缓存。
- 没有明确时刻、「顺便看一眼」性质的巡检 → **heartbeat**。看看目录有没有新文件、提醒休息、检查待办。

务实的分界线：**凡是「漏一次你会生气」的任务，用 cron；凡是「多跑几次无所谓、不跑也行」的任务，才放进 heartbeat。**

## 配置做法

**heartbeat 三步：**
1. 工作区根目录建 HEARTBEAT.md，每条一行，写清判断条件和动作；
2. 配置里调 `intervalSeconds` 和 `activeHours`（比如只在 09:00–23:00 跑）；
3. 观察几天，把从没触发过动作的条目删掉或挪去 cron。

**cron 三步：**
1. `openclaw cron add --name daily-report --cron "0 9 * * *" --tz "Asia/Shanghai" --session isolated --message "汇总…"`（参数以你的版本为准）；
2. 要推结果就加 announce/投递参数，不想污染主会话上下文就用 isolated 会话；
3. `openclaw cron list` 确认注册，`openclaw cron runs` 看历史和报错。

## 踩坑点

1. **把定时日报写进 HEARTBEAT.md**。模型不是每次心跳都会「想起来」执行，不稳定还费 token。定时类一律 cron。
2. **心跳间隔调太小**。每次心跳都是一次真实模型调用，5 分钟一跳、清单又长，成本和噪音都会上来。清单控制在几行以内。
3. **cron 忘了设时区**。「早上 9 点」可能按网关所在时区变成 UTC 9 点，加 `--tz`。
4. **cron payload 太瘦**。隔离会话里模型没有你的上下文，message 要把目标、数据源、输出格式写全，否则空转。
5. **两边重复**。心跳清单里留着 cron 已在干的活，会收到重复通知。定期拿 HEARTBEAT.md 和 `cron list` 对一遍。
6. **指望补跑**。机器关机期间两者都不会补执行，关键任务的前提是网关常驻在线。

## 可复用建议

- HEARTBEAT.md 当「巡检清单」而不是「任务队列」，条目必须有明确判断条件；
- 确定性调度全部收敛到 cron，统一 `list/runs` 管理，出问题有历史可查；
- 心跳用 activeHours 限时段，夜间不跑，省钱也安静；
- cron 统一走 isolated 会话 + announce 投递，主会话保持干净；
- 每月复盘一次：该从 heartbeat 晋级为 cron 的就晋级，没价值的条目直接删。

## 总结

cron 和 heartbeat 不是竞争关系，而是两层：**cron 负责「准时」，heartbeat 负责「自觉」**。确定性交给 cron，模糊巡检留给 heartbeat，并保持心跳清单足够小。这套组合稳定跑下来后，改动频率远低于当初混用的阶段——选对工具，比调参管用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/1eac0f8c5988635a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/09443fdbb5f59dd4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/e975bb89be2cc30e.png)

