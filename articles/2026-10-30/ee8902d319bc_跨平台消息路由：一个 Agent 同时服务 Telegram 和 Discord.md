---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 42976
source: 综合讨论
publishedAt: 2026-10-30
---

## 背景

我们团队和社区用户分散在两处：日常沟通在 Telegram 群，项目讨论和公告在 Discord 服务器。之前是两个独立 bot，问题很明显：记忆不互通、prompt 和 skills 维护两份、同一个人换个平台 Agent 就"失忆"，修个问题还要改两处。

目标很朴素：跑一个 Agent 实例，两个平台只是两个入口，能力和记忆完全一致。

## 问题拆开看

真正要解决的只有三件事：

1. **会话归属**：同一个用户从 TG 和 Discord 进来，上下文怎么算？
2. **消息形态**：两边的 markdown 方言、长度上限、编辑与回复行为都不同。
3. **运行时**：双通道并发、平台限流、长任务阻塞主循环。

## 做法

**1. 单 Gateway，双通道，单 Agent**

OpenClaw 的架构天然适合这个场景：Gateway 收敛消息入口，通道只是适配器。一份配置同时声明两个通道，agent 绑定同一实例：

```jsonc
{
  "channels": {
    "telegram": { "botToken": "TG_TOKEN", "dmPolicy": "pairing" },
    "discord":  { "token": "DC_TOKEN", "intents": ["guildMessages"] }
  },
  "agents": { "default": { "workspace": "~/agent-workspace" } }
}
```

**2. 会话按"通道 + chat id"隔离**

默认 session scope 设为 per-chat，两个平台互不串台，排查问题也干净。需要跨平台连续性（比如用户在 TG 发起任务、回 Discord 看结果）时，靠共享 workspace 文件和长期记忆承接，而不是共享会话——实践下来这样最不容易出诡异 bug。

**3. 身份映射**

建一张内部 user 表：平台 user id → 内部用户。权限、记忆、配额都挂在内部用户上，通道只是登录方式。

**4. 出站渲染下沉**

Agent 内部只产出一份规范 markdown，每个通道配一个很薄的 renderer：处理转义（TG 用 HTML parse_mode 省心）、分片（Discord 2000 字符、TG 4096）、@提及格式差异。**不要**让平台语法混进 agent 上下文。

**5. 出站队列 + 异步长任务**

每通道一个带指数退避的发送队列；长任务先回一条 ack，算完再推送结果，避免阻塞主循环拖死另一边。

## 踩坑点

- **Telegram 409 conflict**：两个进程同时 polling 就会报错。webhook 和 polling 只能选一个，重启前确认旧进程清干净。
- **Discord 收不到消息正文**：MESSAGE CONTENT intent 没开时，bot 只能看到空消息，很容易误判为 Agent 出错，先查后台开关。
- **TG 编辑限流**：流式输出高频 editMessageText 会触发 429，按 1.5–2 秒间隔合并编辑；Discord 则用 typing 状态 + 一次性发送。
- **DM 与群聊 session key 撞车**：同一用户私聊和群聊要分 session，否则上下文互相污染。
- **媒体路径**：两边下载限制不同，统一落盘到 workspace 固定目录，agent 只引用路径。

## 可复用建议

- 通道适配器保持"薄"，所有业务逻辑进 agent/skills。将来加第三个平台（比如 Slack）只是多一个 adapter。
- 先设计统一消息信封：文本、引用、附件、回复目标、优先级，渲染器只消费信封。
- 每通道独立 feature flag，一个通道故障不影响另一个。
- 录几条真实入站消息做 fixture，升级前回放做回归。
- 监控按通道拆：发送失败率、端到端延迟、429 次数，出问题一眼定位是哪边。

## 总结

跨平台路由的工程量不大，难点在于把平台差异**挡在渲染层和队列层**，让 Agent 完全不感知"自己挂在哪个平台"。原则一句话：单 Agent、多通道、会话隔离、渲染下沉。跑稳之后，加新通道基本是纯增量工作。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/fc2233d684342ca0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/f72f63d9467b3bc7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/0d9598169bade5cd.png)

