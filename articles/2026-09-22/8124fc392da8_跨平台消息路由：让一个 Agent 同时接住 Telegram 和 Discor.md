---
title: 跨平台消息路由：让一个 Agent 同时接住 Telegram 和 Discord
feedId: 38412
source: 综合讨论
publishedAt: 2026-09-22
---

## 背景

我的 Agent 最初只挂在 Telegram 上，后来团队日常协作迁到了 Discord，于是出现了两套 bot 进程、两份会话历史、两份工具配置。同一件事在 Telegram 交代过，换到 Discord 它一无所知。与其维护两个 Agent，不如让一个核心同时服务两端。

## 问题

拆开看其实是三件事：

- 双实例导致记忆与 MCP 工具上下文分裂，配置要维护两遍；
- 两个平台的消息格式、长度上限、速率限制完全不同；
- 同一个自然人在两个平台，算一个会话还是两个会话，需要明确决策。

## 做法

核心原则：**Agent 逻辑只写一份，平台差异全部压进连接器层。**

配置骨架大致是：

```yaml
channels:
  telegram: { mode: polling }
  discord:  { mode: gateway, intents: [message_content] }
router:
  session_key: "{platform}:{user_id}"
  shared_memory: summary
limits:
  telegram: { out: 1/s, max_len: 4096 }
  discord:  { out: 5/5s, max_len: 2000 }
```

1. **单进程双连接器。** Telegram 用 long polling 免去公网回调，Discord 走 gateway websocket。连接器只做收发与格式转换，不碰业务逻辑。
2. **定义统一消息信封**：`platform / channel_id / user_id / reply_to / attachments / raw_text`，核心只认信封字段。
3. **会话路由**按 `(platform, user_id)` 各开 session，通过共享 memory store 同步对话摘要。同一自然人的双平台身份先不合并，跑两周再决定。
4. **出站各挂一个 renderer**：Telegram 走 MarkdownV2，Discord 走原生 markdown；超长回复按平台上限切分，切分时保护代码块完整。
5. **每平台一个限速出站队列**，超速排队而不是硬发。

## 踩坑点

- Discord 后台没开 MESSAGE CONTENT intent 时，收到的消息内容恒为空，症状极具迷惑性，排障先查这个。
- MarkdownV2 转义表很长，漏一个下划线就是 400 错误。稳妥做法是对非代码文本全量 escape。
- 长回复按字符数硬切会切断代码块，两个平台的渲染会一起崩，要按块切。
- 千万别让两端 bot 在同一个群里互相转发，会形成回声循环——我的 Agent 曾和自己聊了一整晚。
- 工具执行慢时先发 typing / chat action，否则用户默认它已经死了。

## 可复用建议

- **连接器薄、核心厚**：连接器超过两百行就该警惕是不是混进了业务判断。
- 信封 schema 先定稳，之后接入第三个平台只需新增连接器。
- 日志始终带 platform 标签，排障时按平台过滤能省一半时间。
- 先 polling 跑通全链路，再考虑切 webhook 优化延迟。

## 总结

跨平台不是"多写一个 bot"，而是一次把消息进出与 Agent 核心彻底解耦的机会。做完之后最大的收益不是少跑一个进程，而是记忆、工具与人格配置终于只有一份。按这套结构，接入第三个平台（比如 Slack）预计一个晚上能搞定。欢迎在评论区交流你们的消息信封设计和会话合并策略。

---

