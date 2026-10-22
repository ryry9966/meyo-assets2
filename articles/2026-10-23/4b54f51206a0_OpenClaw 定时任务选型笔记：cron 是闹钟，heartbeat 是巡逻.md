---
title: OpenClaw 定时任务选型笔记：cron 是闹钟，heartbeat 是巡逻
feedId: 42320
source: 综合讨论
publishedAt: 2026-10-23
---

## 背景

用 OpenClaw 做自动化一段时间后，几乎每个人都会遇到同一个困惑：它有两套"定时"机制。**heartbeat（心跳）**让 agent 周期性醒来，扫一眼工作区的 `HEARTBEAT.md`，自主判断有没有值得做的事；**cron（定时任务）**则用 cron 表达式在精确时刻触发一段固定 prompt。两者都能"周期性唤醒 agent"，但设计意图完全不同，混用的代价是时间不准、token 超支和消息轰炸。

## 问题

社区里反复出现的三个提问：

- "我想每天早上 8 点收到日报摘要，用 heartbeat 写进 HEARTBEAT.md 行不行？"
- "heartbeat 默认 30 分钟醒一次，是不是可以替代 cron？"
- "为什么用 heartbeat 做定时后，token 消耗比预期高好几倍？"

根本原因是没分清两种机制的本质：**heartbeat 是巡逻，cron 是闹钟。** 巡逻是"到了就看看有没有事"，闹钟是"到点必须做这件事"。

## 做法：三问选型

配置前先问自己三个问题：

1. **时间要精确吗？** 每天整点发日报、每周一跑备份 → cron。
2. **需要主会话上下文吗？** 任务依赖你之前和 agent 聊过的内容 → heartbeat（天然走主会话），或 cron 的 main 会话模式。
3. **是"看情况做"还是"到点必做"？** 环境感知类（检查收件箱、盯状态变化）→ heartbeat；确定性任务 → cron。

**cron 配置要点：** 定义 job 时写清 cron 表达式、prompt、会话模式（isolated/main）和投递目标。推荐 isolated 模式跑确定性任务，避免污染主会话；此时 prompt 必须自包含，当成一个独立函数来写：输入、输出、投递到哪，一样不缺。

**heartbeat 配置要点：** `HEARTBEAT.md` 只放"检查项"，不放重活；间隔别低于 30 分钟起步；和 agent 明确约定"无事返回 HEARTBEAT_OK 保持沉默"，这是控制噪音和成本的关键。

## 踩坑点

1. **把 heartbeat 当 cron 用。** "每天 8 点发日报"写进 HEARTBEAT.md，agent 只能在唤醒周期附近触发，时间不准，且每次唤醒都烧一遍主会话上下文的 token。
2. **时区坑。** Docker 容器默认 UTC，cron 写的 8 点实际是北京时间 16 点。显式设置 `TZ=Asia/Shanghai`，别靠猜。
3. **isolated 会话丢上下文。** cron prompt 里写"继续之前的分析"，隔离会话根本不知道"之前"是什么。要么 prompt 写全，要么改 main 模式并接受上下文污染。
4. **heartbeat 输出失控。** HEARTBEAT.md 里检查项写得含糊，agent 每次都觉得"有事要汇报"，一天几十条消息。沉默条件必须写死。
5. **僵尸 cron job。** 试验性任务建完不删，半夜悄悄触发、悄悄烧 token。定期审计。

## 可复用建议

- **默认策略**：确定性任务一律 cron + isolated；只有"需要主会话记忆的主动性检查"才用 heartbeat。
- **算清成本**：heartbeat 开销 ≈ 唤醒次数 × 单次上下文 token。先拉长间隔跑两天，看账单再收紧。
- **prompt 即函数**：每条 cron prompt 独立可运行，不依赖对话历史。
- **月度审计**：列出全部 cron job 和 HEARTBEAT.md 内容，删掉不再需要的——定时任务是那种"建完就忘"的典型。

## 总结

cron 和 heartbeat 不是竞争关系，是分工：**cron 管确定性、heartbeat 管环境感知。** 选错机制的代价不只是不准时，还有 token 账单和消息噪音。下次建定时任务前，先问一句"它需不需要主会话上下文、时间要不要准"，答案基本就出来了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/7d8571bb945258a9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/73bc2e113df5b726.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/bb46d1ec7eb238c4.png)

