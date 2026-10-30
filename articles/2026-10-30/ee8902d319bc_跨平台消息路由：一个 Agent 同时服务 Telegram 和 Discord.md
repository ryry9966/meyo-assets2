---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 42993
source: 综合讨论
publishedAt: 2026-10-30
---

# 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord

## 背景

我们的 OpenClaw agent 最初只接了 Telegram，个人和小群用着没问题。后来团队协作搬到 Discord，割裂就出现了：同一批人要在两个平台分别 @ 两个 bot，agent 的记忆、工具和任务状态各一份。目标很朴素——一个 agent 实例、一份记忆，两个入口都能用，必要时还能跨平台传话。

## 问题

拆开看是四件事：

1. **接入**：Telegram 走 long polling，Discord 走 gateway websocket，两套事件模型；
2. **会话归属**：消息进来后路由到哪个 session，不同群、不同人不能共享上下文；
3. **格式差异**：Markdown 方言、长度上限（Telegram 4096 / Discord 2000）、附件、引用回复各不相同；
4. **回程路径**：agent 的回复和工具产出的消息，怎么决定发回哪个平台。

## 做法

### 1. 双通道接入，adapter 归一化

gateway 同时挂两个 channel，token 各自申请，两个 adapter 并行跑。关键是业务逻辑不感知平台差异，归一化在 adapter 层完成：

```yaml
channels:
  telegram:
    enabled: true
    mode: polling
  discord:
    enabled: true
    intents: [guild_messages, dm_messages]
```

统一后的内部消息模型：`{ channel, chat_id, thread_id?, sender, text, attachments[], reply_to }`。

### 2. Session 路由

session key 用 `(channel, chat_id, thread_id)` 三元组，同一个 agent，不同 key 不同上下文：

- Telegram 群 A 和 Discord 频道 #dev 是两个独立 session；
- Discord 的每个 thread 单独成 session，避免话题互相污染；
- DM 走 `(channel, sender)` 作为 key。

### 3. 回程路由

回复默认回原路（reply-to-source）。跨平台发送做成显式的 MCP 工具，比如 `send_message(channel, target, text)`，让模型在明确意图时才调用，而不是隐式广播。

### 4. 出站统一适配

出站统一走一个 dispatcher：按目标平台重排 Markdown、超长自动分段、附件转链接。分段逻辑放 dispatcher 而不是各 adapter，保证两端行为一致。

## 踩坑点

- **自激循环**：agent 在 Discord 发的消息会被自己的 gateway 再次捕获。必须在入口过滤自身 bot id，以及被 agent @ 到的其他 bot 的纯回显。
- **Markdown 转义**：Telegram MarkdownV2 的转义规则极严格，`_`、`[`、反引号都可能直接 400。我们最终放弃 MarkdownV2，统一走 HTML 子集。
- **429 退避**：Discord 短时间多分段容易触发限流，dispatcher 里加了全局令牌桶，并尊重 retry_after。
- **群聊上下文污染**：群里所有闲聊都进 session 会很快炸掉。加两层过滤——只响应 @提及/回复 bot 的消息，其余作为低权重背景记录。
- **附件时效**：Discord CDN 链接有签名时效，第二天就失效。转存到自己的对象存储后再交给 agent。

## 可复用建议

1. **先做消息归一化，再加通道**。adapter 只做翻译，业务永远面对统一模型，以后接新平台就是新写一个 adapter。
2. **session key 三元组是这套架构里最值钱的决定**，上线至今没为路由改过业务代码。
3. **跨平台动作必须显式**。隐式同步（A 平台消息自动搬到 B）我们试了一周就关掉：噪音大、上下文膨胀、用户不信任。显式工具调用才可控。
4. **路由决策留日志**。`msg_id → session_key → reply_channel` 一条链路日志，排障时间少一大半。
5. 灰度路径：先做单向推送（摘要搬运），稳定后再放开双向和主动消息。

## 总结

一个 agent 服务多平台，难点不在"接上"，而在路由与格式这两层基础设施。adapter 做薄、session key 设计好、跨平台动作显式化，这三件事做对，剩下的就是常规运维。目前这套结构正在接入第三个通道，新增成本基本就是写一个新 adapter。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/1c0e53bfa98ef89b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/d6124b9a2c685657.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/7d54ec27e2078014.png)

