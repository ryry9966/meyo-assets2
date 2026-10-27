---
title: cron 还是 heartbeat？OpenClaw 定时任务的选型笔记
feedId: 42747
source: 综合讨论
publishedAt: 2026-10-27
---

## 背景

OpenClaw 里有两套"定时唤醒"机制，新用户经常混着用，然后被结果困惑：

- **cron**：传统意义上的计划任务。用 cron 表达式定义"什么时候跑"，每次执行带独立的提示词，可以跑在隔离会话里，结果可投递到指定渠道。
- **heartbeat**：心跳。Agent 每隔 N 分钟被唤醒一次，读工作区里的 `HEARTBEAT.md`，自己判断有没有事要做，没事就静默（约定俗成的 `HEARTBEAT_DONE`）。

两者本质都是"到点了给模型塞一段 prompt"，但设计意图完全不同：一个解决"几点做"，一个解决"出事再做"。

## 问题

我见过两种典型误用：

1. 把"每天 8 点发晨报"写进 `HEARTBEAT.md`，指望心跳准点触发——心跳间隔是轮询节奏，不是闹钟。
2. 把"盯着收件箱，有急事就叫我"拆成七八个 cron job——条件类需求用 cron 表达式根本表达不了，最后变成每 5 分钟一次的火力覆盖。

## 做法

### 时间驱动 → cron

```bash
openclaw cron add --name morning-brief \
  --cron "30 7 * * 1-5" \
  --tz Asia/Shanghai \
  --session isolated \
  --message "拉取我的 GitHub 通知和今日日历，输出 200 字以内晨报" \
  --channel telegram --to "@me"
```

管理用 `cron list / runs / enable / disable`，临时验证用 `cron run`。三个要点：prompt 自包含（隔离会话没有你的聊天上下文）、明确投递目标和输出格式、显式设时区。CLI 参数偶有变动，以 `openclaw cron add --help` 为准。

### 状态驱动 → heartbeat

`HEARTBEAT.md` 写成可判定的 checklist，不要写散文：

```markdown
# Heartbeat
- 收件箱有未读且标记紧急的邮件 → 立即通知我
- ~/downloads 堆积超过 2GB → 提醒清理
- 其余情况：无事
```

间隔在配置里调，默认 30 分钟：

```json
{ "agents": { "defaults": { "heartbeat": { "every": "30m" } } } }
```

心跳跑在主会话里，共享上下文，这是双刃剑：它"知道你最近在聊什么"，但每次 tick 都消耗 token。

### 组合拳

实战里最好用的是组合：cron 负责确定性的"拉"，heartbeat 负责低频的"盯"。比如 cron 每天 9 点拉监控数据写进本地文件，heartbeat checklist 里加一条"对比昨日指标，波动超 20% 才报"。数据通过文件流转，两个机制各干各的。

## 踩坑点

1. **心跳不准点**。间隔是轮询节奏，网关重启、机器睡眠都会漂移。所有"X 点整必须发生"的需求一律 cron。
2. **HEARTBEAT.md 太长**。每个 tick 都会带上它，三页散文等于给自己上 token 税，保持在一屏内。
3. **心跳噪音**。务必让 agent 无事时走静默约定，而不是每次都汇报"我检查过了，没事"。通知渠道也收窄到一个。
4. **cron 隔离会话"失忆"**。job 的 message 要像给新同事的一次性工单：工具、路径、输出格式写清楚，别指望它记得你上周的话。
5. **时区**。服务器在 UTC、人在东八区，晨报秒变下午茶。
6. **心跳间隔别贪快**。调到 5 分钟省不了事，主会话上下文很快被心跳记录填满，反而劣化日常对话质量。

## 可复用建议

选型一句话：**"几点做"选 cron，"出事再做"选 heartbeat；两头都要，cron 生产、heartbeat 消费。**

- 新需求先问一句：触发条件是时间点还是状态？
- cron 的 message 按"一次性工单"标准写，跑一次 `cron run` 验证再上正式排期。
- HEARTBEAT.md 按"值班巡检表"标准写，条目做减法。
- 每月 `cron list` 清一次僵尸 job，`cron runs` 回看失败记录。

## 总结

cron 和 heartbeat 不是竞争关系，是分工。cron 是闹钟，heartbeat 是巡逻。把精确性交给 cron，把模糊的守望交给 heartbeat，再让数据通过文件或消息在两者之间流动，定时自动化这一层就基本稳了。先跑通一个最小组合（一个 cron job + 三条心跳 checklist），再逐步加码，比一上来堆十个任务靠谱得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/5a46aea9798fa13f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/d54474be2e6c863f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/eb247bf838956d37.png)

