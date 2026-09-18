---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 38058
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

OpenClaw 是常驻运行的 Agent 网关，定时能力有两条完全不同的路径：**cron** 负责精确的时间表触发，**heartbeat** 负责周期性的"醒来看看有没有事"。我在社区里见过最多的两类误用：把"每天 9 点推日报"写进 HEARTBEAT.md，结果时间不可控；把"盯着邮箱有新邮件就提醒我"配成 cron，结果要么漏跑要么重复。这篇把两者的边界讲清楚。

## 问题：驱动方式不同，语义就不同

- **cron**：外部时间驱动。到点执行一次，支持独立会话、指定渠道投递，执行完即结束。
- **heartbeat**：内部周期驱动。Agent 每 N 分钟醒来读一遍 HEARTBEAT.md，自己判断做事还是保持安静（无事时只回一个 OK）。

一句话：**cron 是闹钟，heartbeat 是反射弧。**

## 做法

**1. 固定时间的任务用 cron：**

```bash
openclaw cron add --name morning-report \
  --cron "0 9 * * *" \
  --message "汇总昨日 commit 与待办，发到 Telegram" \
  --tz Asia/Shanghai
```

关键点：显式指定时区；确认投递模式是 announce 还是 send；重任务跑在独立会话里，不污染主上下文。

**2. 条件触发的巡检用 heartbeat：**

在配置里调 `HEARTBEAT_INTERVAL_MS`（默认 30 分钟，建议 15–60 分钟之间），然后把 HEARTBEAT.md 写成**条件清单**而不是命令清单：

```markdown
- 若 GitHub 有新的 issue mention，摘要并提醒
- 若磁盘使用率超过 85%，提示清理
```

Agent 每次心跳只处理满足条件的项目，否则静默。

## 踩坑点

1. **时区**：cron 默认按宿主机时区跑，服务器在 UTC 时"早 9 点"会变成下午 5 点，务必显式 `--tz`。
2. **HEARTBEAT.md 写成命令**："每小时总结一次新闻"这类条目会让 Agent 每次心跳都执行——频率由间隔控制，不由条目文本控制。
3. **重活挂 heartbeat**：构建、爬取这类长任务会阻塞下一次心跳，输出堆积进上下文后 token 消耗明显上涨。重活交给 cron + 独立会话。
4. **cron 不补跑**：fire-and-forget，触发时刻进程恰好挂了这一次就丢了。关键任务加一条心跳兜底："若昨日定时任务未执行则补做"。
5. **静默失败**：cron 投递目标渠道未连接时不报错，先用 `openclaw cron list` 加手动 run 一次验证链路。

## 可复用建议

- 决策口诀：**有确定时间 → cron；只有条件没有时间 → heartbeat**。
- HEARTBEAT.md 控制在 5 条以内，全部写成"若 X 则 Y"。
- 心跳间隔别低于 15 分钟，先观察一周 token 用量再调。
- 需要"补跑"语义的用 heartbeat 兜底，需要"准点"语义的用 cron。
- 两者可以组合：cron 负责产出定时结果，heartbeat 负责异常巡检。

## 总结

cron 和 heartbeat 不是替代关系，而是两种触发语义：一个对时间负责，一个对状态负责。把"何时做"交给 cron，把"要不要做"交给 heartbeat，Agent 的自动化才会既准时、又安静、还不烧钱。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/12aa126cf750220a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/46fc9ba2b1e052a9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/a8e3fc895f785e3b.png)

