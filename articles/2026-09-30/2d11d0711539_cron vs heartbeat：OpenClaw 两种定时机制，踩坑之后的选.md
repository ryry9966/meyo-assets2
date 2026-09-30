---
title: cron vs heartbeat：OpenClaw 两种定时机制，踩坑之后的选型笔记
feedId: 39836
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

OpenClaw 里有两套"让 agent 自己动起来"的机制，新人很容易混着用：

- **cron**：由 `openclaw cron` 管理的定时任务，标准 cron 表达式，到点就跑。跑什么 prompt、结果投递到哪个频道，都在任务定义里写死。
- **heartbeat（心跳）**：agent 按固定间隔被自动唤醒，读一遍 workspace 里的 `HEARTBEAT.md`，由模型自己判断"这一轮有没有事要做"。没事就回 `HEARTBEAT_OK`，静默结束，几乎不产生推送和成本。

两者走的是同一个 agent loop，但设计意图完全不同。

## 问题

我最初的用法很典型：全塞给 heartbeat。把"每天早上发日报""周五提醒周报"全写进 HEARTBEAT.md，指望 agent 到点自觉执行。结果是日报时灵时不灵——模型有时觉得"今天好像没什么可报"就跳过了。反过来，也有朋友把心跳间隔压到 5 分钟当轮询用，一周下来 token 消耗非常难看。

本质是：**cron 是确定性调度，heartbeat 是概率性巡检**。拿错工具，要么漏任务，要么烧钱。

## 做法

我现在的分工原则：

**cron 管"有明确时刻"的事：**

```bash
openclaw cron add --name daily-report \
  --cron "0 9 * * *" \
  --prompt "汇总昨天 workspace/notes 的变更，生成日报发到 Telegram"
```

到点必跑，跑在 isolated session 里，不污染主对话上下文。

**heartbeat 管"条件触发"的事。** HEARTBEAT.md 里只写巡检项，不写任务队列：

```markdown
# 巡检清单
- inbox/TODO.md 有新增未处理项吗？有则整理并提醒
- 等跟进的那封邮件有回复吗？有则摘要给我
```

agent 每次醒来扫一遍，绝大多数时候以 `HEARTBEAT_OK` 结束，成本趋近于一次短 prompt。

**组合用法**：cron 负责准点拉数据落盘（写进 workspace 文件），heartbeat 负责看文件决定要不要打扰你。确定性调度 + 条件判断，各干各的。

心跳间隔在配置里可调（默认 30 分钟），按需放宽或收紧。

## 踩坑点

1. **HEARTBEAT.md 写成任务队列**。堆积几十条待办后，每次心跳全部进上下文，token 翻倍还容易误执行。它是巡检清单，不是 TODO 应用。
2. **把心跳输出投递到群里**。配置成每次心跳都推送的话，`HEARTBEAT_OK` 之外的碎碎念会刷屏。只允许"有实际产出"时投递。
3. **cron 时区**。服务器默认 UTC，你写的"每天 9 点"实际是北京时间 17 点。部署后先手动触发一次核对时间。
4. **心跳间隔压太短**。低于 10 分钟基本是纯烧钱，除非你明确知道自己在轮询什么高频状态。
5. **长任务和心跳撞车**。心跳触发的长任务会和正常会话抢并发，跑批类的事一律放 cron。

## 可复用建议

- 一句话选型：**时刻明确 → cron；只关心状态 → heartbeat**。
- 新增需求时先问自己："错过一次会怎样？"不能错过 → cron；晚半小时无所谓 → heartbeat。
- cron 任务统一 isolated session + 明确投递目标，输出可追溯。
- HEARTBEAT.md 保持一屏以内，每两周结合 token 消耗做一次瘦身，顺便倒推心跳间隔是否合理。

## 总结

cron 和 heartbeat 不是替代关系，而是互补：cron 保证"该发生的事准时发生"，heartbeat 保证"值得注意的事不被漏掉"。把确定性的交给调度器，把判断性的交给模型，OpenClaw 的自动化才能既省心又省钱。有别的组合玩法欢迎在评论区交流。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/800875854805b45e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/d40b76794a3d3c88.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/8ac49eca97bcdb97.png)

