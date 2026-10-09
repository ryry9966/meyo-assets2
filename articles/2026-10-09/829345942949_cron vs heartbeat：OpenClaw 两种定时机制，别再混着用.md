---
title: cron vs heartbeat：OpenClaw 两种定时机制，别再混着用
feedId: 41018
source: 综合讨论
publishedAt: 2026-10-09
---

## 背景

OpenClaw 里有两套"定时"机制，新手很容易混用：

- **cron**：网关内置的调度器，标准 cron 表达式，到点就跑一个明确的任务，可以把结果投递到指定 channel；
- **heartbeat**：agent 的心跳循环，默认每 30 分钟醒来一次，读取 `HEARTBEAT.md` 里的清单，由模型自己判断"有没有事要做"，没事就回 `HEARTBEAT_OK` 保持沉默。

本质区别一句话：cron 是你下指令它执行，heartbeat 是它定期自检。

## 问题

我最初的用法是错的：把所有周期性需求全塞进 `HEARTBEAT.md`——早报、每周 issue 汇总、每小时查磁盘。结果两个问题：

1. **token 成本不可控**。心跳每次都是一次完整的 agent turn，清单越长，每次醒来烧得越多，高频心跳更是雪上加霜；
2. **行为不可靠**。"模型自己判断"意味着偶尔漏做、多做，或在错误的时间做。每天早报这种需求，抖动半小时不可接受。

## 做法：按"确定 vs 感知"分流

判断标准一句话：**时间点和动作都确定的任务用 cron；"看看有没有异常"类的巡逻交给 heartbeat。**

cron 侧示例：

```json
{
  "name": "morning-digest",
  "schedule": { "kind": "cron", "expr": "0 8 * * *" },
  "payload": "汇总昨天的未读消息和日历，生成日报",
  "deliver": true,
  "channel": "discord"
}
```

要点：任务跑在隔离 session 里，别污染主会话上下文；一次性任务用 `kind: "at"` 指定具体时间，跑完自动清理；投递目标写明确，别让模型猜。

heartbeat 侧，`HEARTBEAT.md` 只放"巡逻项"，不放"定时任务"：

```markdown
# Heartbeat
- 收件箱中标记"紧急"且超过 2 小时未回的邮件，提醒我
- ~/sync 目录出现冲突文件时告诉我
其他情况：HEARTBEAT_OK
```

关键是明确告诉它"其他情况保持沉默"，否则它会找借口说话。

## 踩坑点

1. **心跳频率调太高**。为了"及时"把间隔从 30 分钟压到 5 分钟，一天多出几百次 agent turn，日志和账单都难看。真正需要"及时"是 cron 的活，heartbeat 只兜底。
2. **HEARTBEAT.md 写成需求文档**。塞到 20 条后模型开始漏项。现在控制在 5 条以内，超出一律转成 cron job。
3. **时区没对齐**。容器里是 UTC，`0 8 * * *` 实际对应北京时间 16 点。用 `TZ` 环境变量对齐，或加任务后先观察一轮。
4. **心跳里做了事却没人收到**。heartbeat 的输出默认留在本地会话，要通知人得让它显式调用消息发送，或者干脆改成 cron + deliver。

## 可复用建议

- 选型口诀：**cron 管"什么时候做"，heartbeat 管"要不要做"**。
- `HEARTBEAT.md` 当低 SLA 的监控清单，cron 当真正的调度层。
- 心跳间隔宁长勿短，先按默认 30 分钟跑两周，再按实际漏报收紧。
- 每月用 `openclaw cron list` 清理一次，躺着的一次性任务记得删。

## 总结

两者不是竞争关系，是分工：cron 负责确定性，heartbeat 负责自主感知。把确定性需求从心跳清单里搬出去之后，我的 token 消耗降了约 60%，任务准时率反而上来了。先用好 cron，再让 heartbeat 做兜底巡逻，是更稳的路径。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/aff8a083bd8e510d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/e477c67e04082f78.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/729b732404f7edb4.png)

