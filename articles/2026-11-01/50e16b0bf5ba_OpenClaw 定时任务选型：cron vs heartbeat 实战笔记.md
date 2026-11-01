---
title: OpenClaw 定时任务选型：cron vs heartbeat 实战笔记
feedId: 43168
source: 综合讨论
publishedAt: 2026-11-01
---

## 背景

OpenClaw 网关是常驻进程，很多动作不能靠用户先开口：早报要在固定时间出现，订阅要有人盯着，提醒要准点响。平台提供了两条定时路径：

- **cron job**：标准 cron 表达式驱动，到点即跑，每次独立会话，结果可投递到指定渠道，支持一次性任务；
- **heartbeat**：固定间隔唤醒 agent，让它读一遍 HEARTBEAT.md，自己判断这轮有没有事可做，没事就静默返回。

两条路都能“周期执行”，但设计意图完全不同。

## 问题

我最初把所有周期需求都塞进了 HEARTBEAT.md：早报、巡检、提醒全写在里面，让 agent 每跳自己判断。结果是：心跳清单越写越长，每跳 token 消耗翻倍；早报时间漂移（心跳间隔不保证准点）；某次心跳里跑了长任务，把用户消息堵在队列里。后来逐步迁回 cron，才确认这两个机制不是互相替代，而是分工不同。

## 做法

选型先问三个问题：

1. **时间是硬约束还是软建议？**“必须 08:30 出现”是硬约束；“有空看一眼”是软建议。
2. **要不要主动投递？** cron 支持把结果推到指定渠道；heartbeat 的产出默认留给 agent 自己处置。
3. **失败要不要独立重试与留痕？** cron 每次 job 独立会话，互不污染上下文；heartbeat 出错会顺着下一跳带过去。

典型分工：

**交给 cron**：固定时刻早报、一次性提醒、按明确间隔轮询外部 API。示例（字段名以你当前版本为准）：

```bash
openclaw cron add --name "morning-brief" \
  --cron "30 8 * * *" \
  --message "汇总昨日通知与今日日程，输出简报" \
  --deliver --channel telegram
```

**交给 heartbeat**：条件触发的巡检，比如“RSS 有新条目且没推送过才发摘要，否则静默”。HEARTBEAT.md 保持短、条件化：

```markdown
# 心跳规则
默认静默返回。仅当满足以下条件时行动：
- RSS 有未推送的新条目 → 推一条摘要到 Telegram
- 磁盘占用 > 85% → 通知我
```

**组合用法**：heartbeat 负责“发现”，发现问题后动态创建一个 one-shot cron 去“执行”，把定时动作留给 cron 的确定性调度。

## 踩坑点

- **时区**：Docker 部署时容器默认 UTC，cron 里的 8:30 不会是你想要的北京时间 8:30。要么给容器设 TZ，要么按 UTC 换算写表达式。
- **HEARTBEAT.md 变大杂烩**：每跳都全量评估一遍清单，token 成本线性上涨。原则：没事就静默，条件要可判定。
- **心跳里干重活**：爬全站、跑批量分析这类活放 heartbeat 会占住 agent、阻塞用户消息。重活转 cron 独立会话。
- **cron 静默失败**：任务挂了但没人知道。关键 job 要加投递确认或失败通知。
- **one-shot 残留**：一次性提醒触发完还躺在列表里。用支持完成后自清理的配置，或定期审计 cron 列表。

## 可复用建议

一句话选型：**到点必须做 → cron；没事别吵我、有事才做 → heartbeat。**

再展开一点：

- 精确时刻、主动投递、需要重试留痕 → cron；
- 状态守卫、模糊巡检、产出不确定 → heartbeat；
- 把“做什么”写进 cron 的 message，把“要不要做”写进 HEARTBEAT.md，两份文件各管各的，别混。

## 总结

cron 是闹钟，heartbeat 是巡逻。硬时刻表交给闹钟，软巡检交给巡逻；heartbeat 发现目标后可以反过来调闹钟。想清楚“这是时间问题还是条件问题”，选型基本不会错。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/41cc72b647e8243f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/9ac5bef1e71f1521.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/0235207fcb3c03f7.png)

