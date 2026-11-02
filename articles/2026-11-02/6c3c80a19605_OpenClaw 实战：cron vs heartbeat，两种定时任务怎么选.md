---
title: OpenClaw 实战：cron vs heartbeat，两种定时任务怎么选
feedId: 43218
source: 综合讨论
publishedAt: 2026-11-02
---

## 背景

OpenClaw 有两套“让 Agent 自己动起来”的机制：

- **cron**：经典定时调度，按 cron 表达式在指定时刻触发一次独立的 Agent 运行，跑完可把结果投递到指定频道。
- **heartbeat**：心跳机制，Agent 每隔 N 分钟（默认 30 分钟）在主会话里醒来一次，扫一眼 HEARTBEAT.md 和上下文，自己决定“做事还是继续沉默”。

两者都能做自动化，但设计目标完全不同，混用误用是社区里最常见的故障来源。

## 问题

1. 把需要“看状态”的任务写成 cron：每次触发都是失忆的孤立会话，不知道上次跑到哪。
2. 把“每天 9 点发日报”挂在 heartbeat 上：靠模型“感觉时间到了”，触发时刻不可控。
3. HEARTBEAT.md 越写越长，每次心跳都带着一大段 prompt，token 账单默默翻倍。

## 判断标准

问两个问题就够：

- 触发条件是**时刻**还是**状态**？时刻 → cron；状态 → heartbeat。
- 任务需要主会话的**记忆和上下文**吗？不需要 → cron；需要 → heartbeat。

| 场景 | 选择 |
| --- | --- |
| 每天 9:00 汇总昨日频道消息发群 | cron |
| 周一早上生成周报 | cron |
| 盯长任务进度，卡住才提醒 | heartbeat |
| 定期看收件箱/Issues，有事才说话 | heartbeat |

## 做法

**配 cron**，示例（字段以你当前版本文档为准）：

```bash
openclaw cron add \
  --name "daily-digest" \
  --cron "0 9 * * *" \
  --prompt "读取昨日频道消息，输出五条以内的要点摘要" \
  --deliver --channel telegram
```

要点：prompt 里写清输入来源和输出格式；确认 delivery 目标已配置，否则任务跑了消息发不出去。

**调 heartbeat**：

1. 心跳间隔别压到分钟级，每次心跳都是一次真实的模型调用；
2. HEARTBEAT.md 只写“检查什么、什么条件算有事”，细节指令放文件里让它按需去读；
3. 明确写上“无事可做时保持沉默”，并在支持时配置活跃时段，避免半夜心跳打扰。

**混合模式**：重定时任务交给 cron，heartbeat 只做轻量巡检；同一职责不要两边都挂。

## 踩坑点

- **cron 是孤立会话**：需要“上次进度”的任务，把状态落盘（JSON/SQLite），在 prompt 里指定路径让它自己读，别指望会话记忆。
- **时区**：cron 表达式按网关所在时区解释，跨时区部署先核对，别让日报晚一小时。
- **heartbeat 隐性成本**：月成本 ≈ 每日心跳次数 × 单次心跳 token，HEARTBEAT.md 精简一行，账单差很多。
- **重复投递**：迁移时先删旧调度再建新的，群里同一件事收两遍是最常见的事故。
- **重活别放 heartbeat**：心跳跑在主会话里，一次长 MCP 调用可能把你的交互请求堵在队列后面。

## 可复用建议

- 一句话规则：**有明确时刻 → cron；有状态要看 → heartbeat；拿不准 → 先 cron，痛了再迁。**
- cron 任务名带前缀（`digest-`、`report-`），方便季度清理死任务。
- 心跳维护的是 prompt 的“短”，cron 维护的是表达式的“准”，二者的维护对象不同。

## 总结

cron 是确定性调度器，heartbeat 是自主性节拍器。选择标准不是谁更强，而是任务由“时刻”驱动还是由“状态”驱动。把精确交给 cron，把判断交给 heartbeat，成本和可靠性就都在掌控之内。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/9f0c2763f75622ba.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/8e8366a3be6aa64b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/7ea5c275345e3089.png)

