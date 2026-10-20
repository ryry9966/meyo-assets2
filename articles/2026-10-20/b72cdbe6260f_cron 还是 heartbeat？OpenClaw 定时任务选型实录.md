---
title: cron 还是 heartbeat？OpenClaw 定时任务选型实录
feedId: 42115
source: 综合讨论
publishedAt: 2026-10-20
---

## 背景

OpenClaw 里有两条“到点干活”的路径：**cron** 和 **heartbeat**。cron 是经典定点触发——写好 cron 表达式，到点执行注册的任务；heartbeat 是 agent 的周期性自检——每隔一段时间醒来一次，自己判断这轮“有没有事要做”，没事就返回 HEARTBEAT_OK 继续睡。两者都能做自动化，但设计意图完全不同，混用是新手最常见的 token 浪费来源。

## 问题

实际跑下来，常见的三种误用：

1. 把“每天 9 点发日报”塞给 heartbeat——定点任务靠 agent 自觉，时间漂移严重；
2. 把所有巡检都写成 cron——触发条件稍有变化就空转，任务列表越堆越长；
3. heartbeat 间隔设成 5 分钟，一周后 token 消耗翻倍，而 agent 绝大多数轮次的结论都是“没事”。

## 怎么选

一句话判断标准：**触发条件是“几点”，用 cron；是“什么情况”，用 heartbeat。**

cron 适合确定性任务：定时日报、周期备份、定点抓取。注册示例（通过 cron 工具）：

```json
{
  "name": "daily-report",
  "schedule": { "kind": "cron", "expr": "0 9 * * *" },
  "payload": "汇总昨日频道消息，生成日报并发送到目标群",
  "deliver": true
}
```

heartbeat 适合条件型巡检：未读消息跟进、任务卡点检查、异常兜底。配置示例：

```json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "30m",
        "target": "channel",
        "activeHours": { "start": "09:00", "end": "23:00" }
      }
    }
  }
}
```

推荐的组合姿势：**cron 打主力，heartbeat 做低频兜底**。比如日报由 cron 发，另有一条“每 2 小时检查一次日报是否已发、没发就补”的心跳逻辑，兜住网关离线导致的漏跑。

## 踩坑点

1. **heartbeat 间隔太短**。30 分钟起步，先跑一周看 token 消耗再调；分钟级心跳基本全是空转。
2. **心跳提示词太泛**。“看看有没有要做的”会让 agent 过度发挥、随机发起任务。建议固定成三段式：触发条件 + 满足时的动作 + 不满足时保持静默。
3. **cron 时区问题**。gateway 时区和你的预期不一致时，任务会在半夜触发。表达式里显式带时区，或上线前先用一条一次性任务验证实际触发时刻。
4. **任务不幂等**。cron 和 heartbeat 盯同一件事时会重复通知。任务侧自己做去重，比如执行前先查“今天的日报是否已存在”。
5. **离线错过不补跑**。cron 默认不补错过的时间点，关键任务要么靠 heartbeat 兜底，要么在任务开头自查状态再决定是否执行。

## 可复用建议

- 新需求先问一句：这是闹钟还是值班？定点的事交给 cron，“有没有事”的判断交给 heartbeat。
- 心跳提示词写三段：条件、动作、静默规则，别让 agent 自由裁量。
- activeHours 一定要配，别让心跳半夜吵人。
- 关键 cron 任务都配一条低频 heartbeat 补偿检查，成本极低，能兜住大部分漏跑。

## 总结

cron 是闹钟，heartbeat 是值班。闹钟管定点的事，值班管临场判断。两者不是二选一，而是分工：多数跑得稳的 OpenClaw 实例，都是 cron 负责确定性输出，heartbeat 负责条件型巡检和兜底。让闹钟守时，让值班冷静，定时任务这块基本就不会出乱子。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/3e519c452547033b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/6c8eaca3cf85c596.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/4236e51e16bd32f9.png)

