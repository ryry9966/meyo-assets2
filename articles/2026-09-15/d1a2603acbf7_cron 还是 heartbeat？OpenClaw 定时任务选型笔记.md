---
title: cron 还是 heartbeat？OpenClaw 定时任务选型笔记
feedId: 37639
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

OpenClaw 里有两条让 agent「自己动起来」的路径：**cron** 和 **heartbeat**。新用户很容易混用，结果要么 token 白烧，要么任务不准点。这篇记录一下我的选型逻辑。

- **cron**：显式注册的定时任务，支持一次性（`deleteAfterRun`）和周期任务，带时区，到点把一条消息投给 agent 执行。
- **heartbeat**：全局心跳循环，默认约每 30 分钟唤醒一次 agent，让它读 `HEARTBEAT.md` 自查「有没有值得关注的事」，由 agent 自己判断要不要说话。

## 问题

选型的本质是：**时间驱动，还是状态驱动**。

- 「每天 8:30 发日报」「下午 3 点提醒开会」——时间确定，用 cron。
- 「盯着仓库有没有新 issue，有再叫我」「检查磁盘，异常才说话」——触发条件是「状态变化 + 判断」，heartbeat 更合适。

用 cron 做轮询监控会尴尬（到点要么硬说点什么、要么沉默），用 heartbeat 做准点提醒会迟到（心跳间隔本身不精确）。这两坑我第一批任务里都踩过。

## 做法

### 1. 用 cron 管精确时间

```json
{
  "name": "morning-digest",
  "schedule": { "kind": "cron", "expr": "30 8 * * *", "tz": "Asia/Shanghai" },
  "payload": { "kind": "agentTurn", "message": "汇总昨天的 GitHub 通知，生成日报发给我" },
  "deleteAfterRun": false
}
```

一次性提醒把 `deleteAfterRun` 设为 `true`，跑完自动清理，不占任务列表。

### 2. 用 heartbeat 管环境状态

在 workspace 的 `HEARTBEAT.md` 里写清「检查项 + 沉默规则」：

```markdown
### Heartbeat checks
- 检查收件箱是否有标记紧急的邮件；有则通知，没有保持沉默。
- 检查 ~/projects/ci 状态文件的 mtime；超过 2 小时未更新才提醒。
- 其他情况一律 HEARTBEAT_OK，不输出。
```

关键是显式告诉 agent「**沉默是合法输出**」，否则每次心跳都会汇报点什么。

### 3. 控制间隔与成本

每次心跳都是一次真实的模型调用。监控类场景 30–60 分钟足够；低频但要求准点的任务一律迁到 cron，别写「每 5 分钟心跳看一眼是不是 8:30」这种浪费写法。

## 踩坑点

1. **时区**：cron 任务不写 `tz` 会按宿主机时区跑。服务器在 UTC 时，8 点的日报变成下午 4 点。永远显式写时区。
2. **心跳噪音**：`HEARTBEAT.md` 不写沉默规则，agent 每轮都汇报「一切正常」，一晚上几十条消息。
3. **心跳里塞重活**：长任务会导致心跳排队、周期漂移。重活拆给 cron 或手动触发，心跳只做轻量检查。
4. **一次性任务不清理**：临时提醒忘记开 `deleteAfterRun` 会越积越多，定期 `cron list` 清一遍。
5. **投递通道错配**：任务投到一个你不常看的渠道，等于没投。注册前确认 delivery 目标。

## 可复用建议

- 判断口诀：**到点做事用 cron，出事才说用 heartbeat**。
- 把 `HEARTBEAT.md` 当 SLO 文档写：检查项、阈值、通知条件、沉默条件，四要素齐全。
- 心跳间隔宁长勿短：先 60 分钟跑一周观察噪音率，再逐步收紧。
- 两者可以组合：cron 负责定时拉数据落盘，heartbeat 只读文件做告警判断，把「调度」和「判断」解耦，成本最低。

## 总结

cron 的语义是「我确定这个时刻要做这件事」，heartbeat 的语义是「我授权你定期看看有没有事」。把时间确定性交给 cron，把状态判断交给 heartbeat，成本和噪音都可控。选型错了信号也很明显：**提醒不准点 → 换 cron；消息太吵 → 收紧 HEARTBEAT.md 和心跳间隔**。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/671c891f2321cf01.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/87de5685d6274c14.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/eea0e891d834d3cc.png)

