---
title: OpenClaw 定时任务选型：cron 与 heartbeat 的分工边界
feedId: 43042
source: 综合讨论
publishedAt: 2026-10-30
---

## 背景

OpenClaw 里让 agent“定时干活”有两条路：**heartbeat**，按固定间隔醒来巡检；**cron**，按 cron 表达式定点触发。我一开始混着用，结果既被 heartbeat 的多余消息轰炸过，也遇到过定点任务差了半小时才执行。这篇记录一下摸清两者边界的过程。

## 问题：机制差异到底在哪

| 维度 | heartbeat | cron |
|---|---|---|
| 触发方式 | 间隔（如每 30m） | cron 表达式 + 时区 |
| 运行会话 | 主会话，共享日常上下文 | 默认隔离会话，无主会话记忆 |
| 任务定义 | 工作区的 HEARTBEAT.md，随手改 | 持久化的 job，需增删改管理 |
| 典型用途 | 定期巡视、兜底检查 | 定点投递、固定日程 |

一句话概括：**heartbeat 是定时巡逻，cron 是定点闹钟**。

## 选型判断：问四个问题

1. **要准点吗？**“每天 8:30 准时”走 cron；“每隔一阵看一眼”走 heartbeat。
2. **任务稳定吗？**长期不变的固定任务用 cron；频繁增删改的杂务清单，改 HEARTBEAT.md 一行就行。
3. **需要记忆吗？**依赖主会话上下文的用 heartbeat；任务自包含的，cron 隔离会话更干净。
4. **结果发到哪？**要投递到 Telegram 等指定渠道，用 cron 的 deliver；heartbeat 回复默认走主通道。

## 做法

**heartbeat**：在 `openclaw.json` 配间隔，然后在 HEARTBEAT.md 写巡检项：

```json
{
  "agents": {
    "defaults": { "heartbeat": { "every": "45m" } }
  }
}
```

```md
- 检查 ~/inbox 是否有超过 2 小时未处理的重要事项，有则提醒
- 除确有事项外不要发消息
```

**cron**：最省事的方式是直接在对话里让 agent 建：“每天 8:30（Asia/Shanghai）把未读消息汇总成 200 字摘要发给我”。建完用 `openclaw cron list` 确认任务存在，并手动触发一次验证输出，再放心交给它。

## 踩坑点

1. **时区**：Docker 容器里默认常是 UTC，8:30 会变成 16:30。建 cron 任务时显式带 `tz`，别赌宿主机时区。
2. **把 heartbeat 当闹钟**：它是间隔唤醒，8:00 的事项可能 8:29 才被看到。定点事项一律走 cron。
3. **heartbeat 的噪音与成本**：间隔太短 + 贵模型 = token 消耗飞快。给 heartbeat 单独配个便宜模型，HEARTBEAT.md 措辞上强调“无事勿扰”。
4. **隔离会话没记忆**：cron 的 payload 必须自包含——做什么、发给谁、什么格式，全写清楚，别指望它记得你们昨天聊过什么。
5. **错过不补**：网关宕机错过触发点，默认不补跑（以当前版本行为为准）。重要任务要有兜底。
6. **长任务堆积**：任务耗时超过间隔时可能并发堆积，注意 `maxConcurrentRuns` 或拉长间隔。

## 可复用建议

- **组合拳**：cron 管定点投递，heartbeat 管兜底巡检。比如在 HEARTBEAT.md 加一条“检查今天的日报 cron 是否已发出，没发就补发”，两条机制互为保险。
- **命名规范**：cron 任务统一 `daily-*`、`weekly-*` 前缀，payload 内容存文件维护，别散落在聊天记录里。
- **临时提醒用一次性 cron**（到期即删），别塞进 heartbeat 污染常驻清单。
- **定期体检**：让 heartbeat 每周做一次 cron 巡检——列出任务清单、核对启用状态，防止静默失效。

## 总结

两者不是替代关系，而是分工：**定点、可靠、要投递 → cron；巡视、灵活、靠上下文 → heartbeat**。拿到需求先问两句——“需不需要准点”“需不需要记忆”——答案基本就出来了。把边界划清楚，agent 的自动化才不会变成薛定谔的提醒。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/0b385cec88ab7e01.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/5bdc7f29ef2e8cf0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/13646b9b685afae2.png)

