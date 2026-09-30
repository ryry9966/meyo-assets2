---
title: HEARTBEAT.md：让 Agent 从"被叫才动"变成"定时自查
feedId: 39936
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

用 OpenClaw 一段时间后会发现一个别扭的地方：Agent 的默认交互模式是"你发消息，它响应"。但很多真实需求是反过来的——磁盘快满了要提醒你、日报要定时汇总、项目仓库躺了三天的未提交改动该有人催一下。这些事靠人盯着 trigger，就失去了自动化的意义。

OpenClaw 对此给出的答案是 heartbeat 机制：在 agent workspace 里放一个 `HEARTBEAT.md`，默认每 30 分钟，系统会把这个文件注入主会话做一次"签到"。Agent 读到任务就执行，确认没事就回复 `HEARTBEAT_OK`——这类空转回复会被折叠，不会推送打扰你。

## 问题

heartbeat 把"定时 + 判断 + 行动 + 汇报"的闭环交给了模型，但配不好会翻车：半夜给你发消息、每次心跳空烧 token、同一条告警两小时刷一次屏。根源通常是 `HEARTBEAT.md` 写得像随手贴的便签，而不是一份值守手册。

## 做法

**1. 写任务清单，每条都是"条件 + 动作 + 静默规则"：**

```markdown
## 心跳任务
- 检查 ~/projects 下各 git 仓库是否有超过 3 天的未提交改动，有则提醒，否则回 HEARTBEAT_OK
- 磁盘 / 使用率超过 85% 时通知我，同一告警 6 小时内不重复
- 每 4 小时检查一次 RSS 收件箱，只总结新增条目，无新增则 HEARTBEAT_OK
```

**2. 配置节奏和活跃时段（`~/.openclaw/openclaw.json`）：**

```json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "30m",
        "target": "last",
        "activeHours": { "start": "9:00", "end": "22:00" }
      }
    }
  }
}
```

**3. 用状态文件做跨周期记忆。** 让 Agent 把"上次告警时间""上次已读位置"写进 workspace 下一个小 JSON，下次 tick 读回来，才能实现去重和冷却。

**4. 长周期任务不必靠缩短心跳间隔解决。** 心跳是"巡查 + 决定"，"每天 8 点发日报"这类定点任务交给 cron job 更干净；心跳文件里写清优先级即可。

## 踩坑点

- **任务写得太模糊。**"帮我盯一下事情"会让模型自由发挥，随机打扰你。每条任务必须是模型能明确判定真伪的条件。
- **文件越长，成本越高。** 每次心跳都会消耗 token，即使最终只回 `HEARTBEAT_OK`。建议控制在几十行以内，堆不下的逻辑外移到子文档按需读取。
- **不设 `activeHours`。** 结果就是凌晨三点收到"磁盘使用率 84.7%"的推送。
- **`target` 用 `last` 要留意。** 消息会发到你最近活跃的会话，如果临时在别的渠道聊过天，告警可能发错地方。
- **没有"变化才报"规则。** 告警类任务一定要写明冷却时间和触发条件，否则第一次磁盘报警之后你会收到 48 次同样的提醒。
- **重活别放进心跳。** 心跳执行有超时约束，跑测试、批量抓取这类分钟级任务应该走 cron，心跳只负责发现和分发。

## 可复用建议

- 把 `HEARTBEAT.md` 当值班 SOP 写，而不是 TODO dump。三层分流：能自动处理的执行并记日志；需要决策的推送摘要加一句结论；无事就干净地 `HEARTBEAT_OK`。
- 不同 Agent 不同节奏：监控型 15–30 分钟，日报型只在 activeHours 内触发一两次。
- 先用 60 分钟间隔试运行一周，观察推送质量（有没有废话、有没有漏报），再逐步收紧。心跳是常驻成本，宁可先松后紧。

## 总结

heartbeat 的价值不在"多了个定时器"，而在于把值守逻辑外化成一份可以进 git、可以 code review 的 Markdown。你把它写得越像运维手册，Agent 的行为就越可预测——它应该是一个安静的值班员，而不是一个每半小时找你聊天的话痨。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/f904d33f36494660.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/2b6fe7acae8221d3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/f22c57a30d8efa35.png)

