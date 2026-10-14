---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 41510
source: 综合讨论
publishedAt: 2026-10-14
---

## 背景

OpenClaw 的 agent 是常驻但按需唤醒的，最常用的两种唤醒方式就是 heartbeat 和 cron。新用户的第一个困惑通常是：这俩看起来都是"定时干活"，为什么要做两套？我早期也把所有周期任务塞进 heartbeat，直到 token 账单和延迟问题教我做人，才把分工理清楚。

## 问题

用错机制的代价很具体：把所有任务塞进 heartbeat，间隔一调小，成本翻倍，任务还挤占同一个会话；反过来，把"每天早上 9 点发日报"交给 heartbeat，它既不准点，也没有稳定的投递渠道。

先看机制差异：

| 维度 | heartbeat | cron |
|---|---|---|
| 本质 | 每个 agent 一条周期心跳 | 每个任务一条独立排程 |
| 触发 | 固定间隔，有漂移 | cron 表达式，准点 |
| 上下文 | 共享主会话，延续历史 | 可隔离会话，互不污染 |
| 任务数量 | 一份 HEARTBEAT.md 清单 | 多个 job 并存 |
| 投递 | 会话内回复 | 可 announce 到指定渠道 |
| 空转成本 | 每次心跳都是一次模型调用 | 未触发不花钱 |

## 做法 / 步骤

1. **给任务分类**。问两个问题：需要准点吗？需要独立投递吗？都是"是"→ cron；都属于"帮我盯着"类 → heartbeat。
2. **配 heartbeat**。在 HEARTBEAT.md 写 checklist，每条是一句可判断的指令，比如"检查收件箱是否有需回复的高优邮件，没有就回 HEARTBEAT_OK"。间隔从 30m 起步，别一上来压到 5m。
3. **配 cron**。用 CLI 添加 job，指定 schedule（标准 5 段表达式）、message（payload 指令）、会话策略。重输出型任务建议 isolated 会话 + announce，日报、爬取、汇总都归这类。参数以你版本的 `openclaw cron add --help` 为准。
4. **验证**。`openclaw cron list` 确认任务和下次触发时间；跑一周看日志，核对触发时刻和投递是否如预期。

## 踩坑点

- **heartbeat 的隐形成本是空转**：即使无事可做，agent 也要醒来判断一次。30m 间隔一天约 48 次模型调用，5m 就是 288 次。
- **主会话忙时 heartbeat 会被跳过或延迟**，所以准点类需求千万别放 heartbeat。
- **HEARTBEAT.md 是清单不是 backlog**。见过有人堆二十几条进去，每次心跳全文判断，又慢又贵，过期条目及时删。
- **cron 走 gateway 所在机器的系统时区**，部署在海外 VPS 时先核对，否则"早上 9 点"变成北京时间下午 5 点。
- **隔离会话不免费**：每次触发生成新上下文。日报类任务在 payload 里明确要求"只输出摘要"。
- **gateway 停机期间错过的 cron 不会补跑**，关键任务触发后要确认产出。

## 可复用建议

- 一句话决策：要准点、要多任务、要投递到渠道 → cron；要巡检、要共享上下文、要"顺便看一眼" → heartbeat。
- heartbeat 间隔宁长勿短，先跑一周看 token 消耗再收紧。
- 两者可以组合：heartbeat 负责 15 分钟粒度的环境感知，发现值得持续关注的事，让 agent 顺手创建一条带 announce 的 cron，把临时关注升级成正式任务。
- HEARTBEAT.md 每条控制在一句话以内，能被 HEARTBEAT_OK 快速短路的设计才省钱。

## 总结

heartbeat 是"脉搏"，解决 agent 要不要持续保持警觉；cron 是"闹钟"，解决某件事必须在某时刻发生并送达某处。把巡检给 heartbeat、把准点给 cron，成本和行为都可控。我目前的分工：inbox 巡检和环境监控走 heartbeat（30m），日报、周报、备份提醒走 cron（isolated + announce），运行两个月没再出过调度相关的意外。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/40212a02b1d64338.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/838a97317f8168f4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/618db55564308aad.png)

