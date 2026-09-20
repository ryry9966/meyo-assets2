---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 38292
source: 综合讨论
publishedAt: 2026-09-21
---

## 背景

OpenClaw 的 agent 默认是个被动响应者：你不发消息，它就待着。但很多真正有价值的事情——盯一个目录、检查服务状态、定时汇总——恰恰发生在“你没想到要问”的时刻。

OpenClaw 内置了 heartbeat（心跳）机制：agent 按固定间隔自动醒来一次，读取工作区根目录的 `HEARTBEAT.md`，有可执行任务就执行，没有就返回 `HEARTBEAT_DONE` 并保持沉默。这个文件就是 agent 的“主动性”挂载点。

## 问题

用 cron + 脚本当然也能做，但有三个别扭之处：

- 脚本擅长确定性动作，遇到“判断要不要通知我”这类模糊逻辑很勉强；
- 通知渠道（Telegram / 桌面 / 群机器人）得自己再接一遍；
- 逻辑散落三处，改需求要同时动脚本、cron 和通知配置。

heartbeat 把这些收敛到同一个 agent 会话里：工具、MCP、插件、通知通道它本来就有，`HEARTBEAT.md` 只负责描述**做什么、什么时候开口**。

## 做法

**1. 配置心跳间隔：**

```json
{
  "agents": {
    "defaults": {
      "heartbeat": { "every": "30m" }
    }
  }
}
```

默认 30 分钟，按需调整。

**2. 在工作区根目录创建 `HEARTBEAT.md`**，用“给值班同事写 runbook”的口吻：

```markdown
# Heartbeat

## 每次检查
- 若 ~/reports/ 出现当天新的 *.pdf，提取 3 句以内的摘要发我，然后移走该文件。
- 仅当任务日志连续两次失败时才通知我，否则保持沉默。

## 没事的时候
- 无事可做直接返回 HEARTBEAT_DONE，不要发消息。
```

**3. reload 后观察前几轮心跳日志**，确认触发频率和输出符合预期。

核心写法原则：**每个任务都要有明确的触发条件、动作和沉默条件**。“帮我关注一下 XX”这种模糊任务，会让 agent 每次醒来都“认真分析”一番——token 烧了，没有产出。

## 踩坑点

- **间隔太短 = 烧钱。** 5 分钟一次心跳，即使空转也有固定开销。轻任务 15–30 分钟起步；重任务用文件标记位做“有变化才处理”。
- **污染主会话。** 心跳默认跑在主会话里，中间输出会进上下文。复杂任务要把产出收敛成一句话，或拆给独立 agent。
- **通知轰炸。** 刚配好时 agent 容易“过于勤快”，事事汇报。务必显式写明“无事不发消息”。
- **任务太开放。** “看看有什么新闻”会触发不可控的抓取，应限定数据源（指定 MCP 工具、目录或 RSS 插件）。

## 可复用建议

- 把 `HEARTBEAT.md` 当 **runbook** 维护而非任务堆：纳入 git，每次改动写清动机，每周回顾删掉无效条目。
- 分两层：**高频轻检查**交给心跳；**低频重活**在心跳里只判断触发条件，实际动作交给 cron 调用的 skill。
- 沉默是特性不是缺陷，`HEARTBEAT_DONE` 的价值正在于“不产出价值时零打扰”。

## 总结

HEARTBEAT.md 的本质，是把“主动性”变成一份可版本管理的配置：agent 负责执行和判断，你负责定义什么值得被报告。建议先用一两个低成本任务（比如目录监控）验证，跑稳了再加。主动的 agent 不是更吵的 agent，而是知道什么时候该闭嘴的 agent。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/02d3a0f467efc37e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/aca40fcebf46f96f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/effb0069b4c7c75a.png)

