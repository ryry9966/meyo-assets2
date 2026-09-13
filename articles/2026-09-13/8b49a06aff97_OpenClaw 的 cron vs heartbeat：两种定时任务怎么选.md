---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 37387
source: 综合讨论
publishedAt: 2026-09-13
---

## 背景

OpenClaw 里的 agent 不是"收到消息才醒"的无状态接口，而是常驻进程。既然常驻，就自然有一个问题：怎么让它"自己动起来"？OpenClaw 提供了两条路：

- **cron**：传统意义上的定时任务。预先注册"什么时间点、用什么 prompt、结果发到哪里"，到点由系统唤醒一次执行。
- **heartbeat**：agent 按固定间隔（默认 30 分钟左右）被轻轻拍一下，醒来后读工作区里的 `HEARTBEAT.md`，自己判断现在有没有该做的事。

两者都叫"定时"，但设计意图完全不同。

## 问题

社区里最常见的两类误用：

1. 用 heartbeat 做"每天 9 点发日报"——时间点永远不准，9:07 发出去算好的。
2. 用 cron 做"有空时看看依赖要不要升级"——为一件模糊的事注册精确调度，prompt 里塞不下判断逻辑，结果每天机械触发一次无意义的全量检查。

根因是没分清：**这件事是时间驱动的，还是状态驱动的？**

## 做法

### cron：适合确定性调度

```bash
openclaw cron add \
  --name daily-digest \
  --cron "0 9 * * *" \
  --session isolated \
  --message "汇总昨天的频道消息，生成日报发到 #daily" \
  --deliver
```

要点（具体字段以你版本的 `openclaw cron add --help` 为准）：

- `--cron` 用标准 5 段表达式，先确认调度器时区，别按 UTC 的错觉写时间。
- `isolated` 会开独立会话，跑完即弃；需要哪些上下文，全部写进 `--message`，不要指望它记得你昨天聊了什么。
- 结果通过投递参数发到指定频道，目标要写清楚，否则跑完静默。

### heartbeat：适合周期性巡检

编辑工作区根目录的 `HEARTBEAT.md`（通常在 `~/.openclaw/workspace/` 下），写成 checklist，每条带明确的"跳过条件"：

```markdown
# Heartbeat 任务

- 检查 ~/proj 的 git remote 有没有新分支；
  距上次检查不足 2 小时则跳过，并把检查时间写入 .hb-state
- 收件箱若超过 20 封未读，提醒我；否则什么都不做
```

间隔在配置文件里调整，建议从 `30m` 起步。文件为空或只有注释时，心跳开销很小——这本身就是一个方便的开关。

## 踩坑点

1. **唤醒不等于执行**。heartbeat 每次都会醒来，醒来本身便宜，但指令写得含糊，agent 可能把"看一眼"干成"忙一小时"，token 就烧起来了。每条指令写清楚停止条件。
2. **cron 看不见主会话上下文**。isolated 会话是白纸，"接着刚才那件事"这种 prompt 必然失败。
3. **长任务阻塞队列**。一个跑 20 分钟的任务会把后续触发（包括心跳）排到后面，表现为"莫名延迟"，先查 `openclaw cron list` 和运行日志。
4. **心跳抖动**。"有新版就提醒我"这类规则没有状态，每次醒来都重新判断，可能反复提醒。让 agent 落一个状态文件做去重。

## 可复用建议

一张决策卡：

- 有**确定时间点/日历语义**（每天 9 点、每周一、每 4 小时）→ cron
- 有**模糊条件、需要结合上下文判断**（"有需要再叫我"）→ heartbeat
- 任务要**主动发通知到频道** → cron（投递是内建能力）
- 任务是**自我维护**（巡检、整理、小修小补）→ heartbeat
- 拿不准 → 先用 heartbeat 观察一周，确认稳定高频后再固化成 cron

## 总结

cron 是"到点必须发生"的确定性调度，heartbeat 是"定期醒来自己看着办"的巡检循环。把时间驱动的事交给 cron，把状态驱动的事写进 `HEARTBEAT.md`，两者各司其职，agent 才像一个靠谱的常驻助手，而不是一个要么失联、要么过度热情的定时炸弹。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-13/14607d779cd99dc1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-13/5cb334d7342d7ff4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-13/0b85621551cdcf70.png)

