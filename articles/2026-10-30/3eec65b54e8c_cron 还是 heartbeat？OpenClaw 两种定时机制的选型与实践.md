---
title: cron 还是 heartbeat？OpenClaw 两种定时机制的选型与实践
feedId: 42995
source: 综合讨论
publishedAt: 2026-10-30
---

## 背景

用 OpenClaw 搭常驻 agent，绕不开"定时"这件事：每天早上推日报、每周汇总待办、或者让 agent 自己盯着几个跨天的任务别断线。OpenClaw 提供了两种机制：**cron**（定时任务）和 **heartbeat**（心跳）。新手最常见的第一个坑，就是拿 heartbeat 当 cron 用，或者反过来——结果要么 token 烧得飞快，要么事情总是漏。本帖梳理两者的语义差异，给一个可执行的选型流程和踩坑清单。

## 两者到底差在哪

- **cron**：你指定触发时间和要执行的 prompt，到点就跑，跑完即止。触发是确定的，内容也是你写死的。
- **heartbeat**：agent 按固定间隔被唤醒一次，收到一段心跳 prompt，由它自己判断"现在有没有事要做"。有就干，没有就回 `HEARTBEAT_OK` 静默跳过。

一句话总结：**cron 是闹钟，heartbeat 是值班巡检**。闹钟只在设定时刻响；巡检每班到岗，但到岗后做什么由 agent 自己决定。

## 怎么选：三步判断

1. 触发时间是否固定？固定 → cron。
2. 要做的事是否需要"判断"才能确定？需要判断（检查回复、跟进未闭环事项）→ heartbeat。
3. 结果给不给用户看？给用户、有时间承诺的 → cron；agent 内部维护性质（记忆整理、任务自检）→ heartbeat。

## 实操配置

cron（示例）：

```json
{
  "name": "weekly-digest",
  "schedule": "0 9 * * 1",
  "tz": "Asia/Shanghai",
  "message": "汇总上周会话中未闭环的事项，发到 #daily 频道",
  "deliver": true
}
```

heartbeat（agent 配置）：

```json
{
  "heartbeat": {
    "every": "30m",
    "prompt": "依次检查：1) 是否有等待对方回复且超过 24h 的事项；2) 是否有失败可重试的任务；3) 记忆里是否有今天该记下的结论。若均无，回复 HEARTBEAT_OK，不发送任何消息。"
  }
}
```

## 踩坑点

- **心跳间隔太小**：5 分钟一跳，一晚上 token 消耗非常可观。经验值：无活跃任务 30m–1h，有跨天任务再降到 15–30m。
- **心跳 prompt 太模糊**："看看有什么要做的"会让 agent 自造工作，编出没人要的报告。把检查项写成显式清单，并明确"无事则静默"。
- **cron 时区**：默认按系统时区跑，服务器是 UTC 时日报会下午才发。务必显式写 `tz`。
- **cron 静默失败**：模型调用失败没人知道，任务下次照常触发。给 cron 设投递目标，让结果落到你真会看的频道——异常时至少"该来的没来"能被察觉。
- **同一件事两个 owner**：cron 发了周报，heartbeat 又判断"该发了"再发一份。一个任务只挂在一个机制下。

## 可复用建议

- 选型口诀：**时间固定、内容确定用 cron；时间不定、需要判断用 heartbeat**。
- heartbeat 当看门狗用，别当生产流水线用；重活交给 cron 或显式任务。
- 心跳 prompt 写成"检查清单 + 静默条件"两段式，能显著减少无效唤醒。
- 每周看一次 token 消耗，反推心跳间隔是否合适，这是最直接的调参信号。
- 组合用法：cron 负责"到点开工"，heartbeat 负责"盯着别烂尾"。

## 总结

cron 和 heartbeat 不是替代关系，而是分工关系。把确定性的事交给 cron，把需要判断的事交给 heartbeat，再给 heartbeat 一份明确的检查清单和静默约定——定时系统就会既省 token，又不容易漏事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/52916e2523015fcc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/54e5de91b4c5d178.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/82d845e257c4b72d.png)

