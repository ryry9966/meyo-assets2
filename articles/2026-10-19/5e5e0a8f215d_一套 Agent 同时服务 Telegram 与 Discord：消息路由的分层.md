---
title: 一套 Agent 同时服务 Telegram 与 Discord：消息路由的分层实践
feedId: 42019
source: 综合讨论
publishedAt: 2026-10-19
---

## 背景

用户分散在 Telegram 和 Discord 是社区运维的常态。常见做法是两端各跑一个 bot，各自挂一套 agent，结果很快失控：prompt 维护两份、记忆不互通、修一个 bug 要改两处。我的目标很简单：一个 Agent 实例，两个入口，行为一致。

## 问题拆解

核心矛盾是：Agent 侧希望输入输出是统一结构，而所有平台差异都堆在边缘——

- 消息格式：Telegram 的 MarkdownV2 和 Discord 的 Markdown 不是一套方言；
- 长度限制：Telegram 单条 4096 字符，Discord 2000；
- 接入方式：Telegram 可 long polling 起步，Discord 是 WebSocket 网关长连接；
- 身份体系：同一个人在两端是两个 user_id，会话 key 怎么建。

## 做法

分层很朴素：**Adapter → Router → Agent → Renderer**。

1. **定义中性信封（envelope）**：消息进系统前先归一化，字段包括 `platform / chat_id / user_id / text / attachments / reply_to / message_id`，外加一个 `schema_version`，以后升级不踩自己。
2. **薄适配层**：Telegram 侧用 long polling 起步，Discord 侧走官方库的 gateway。适配层只做翻译，不写业务逻辑。
3. **路由与会话**：session key 用 `platform:user_id`，即默认两端记忆不共享。要不要合并同一人的双端身份，交给用户显式指令触发，别做自动关联（我做错过，别学）。
4. **Agent 保持无感知**：核心只消费 envelope、产出结构化回复，工具调用走 MCP。它不需要知道自己在服务谁。
5. **平台渲染器**：出站统一走 renderer——Discord 按 2000 字符切块（避开代码块中间），Telegram 用 HTML 模式输出。

## 踩坑点

- **别用 MarkdownV2**。十几个字符都要转义，漏一个整条消息直接 400。换 HTML parse mode，问题消失。
- **Discord 读普通消息必须开 MESSAGE CONTENT INTENT**，否则 bot 只能收到 mention，表现为"装死"，我排查了半小时才发现。
- **Telegram webhook 需要公网 HTTPS**。开发期用 polling，上线再切，切换逻辑封在 adapter 里。
- **幂等去重**：Telegram polling 超时重试会重推 update，按 `update_id` 去重，否则同一条消息回两遍。
- **限速差异**：Discord 按 channel 限速，Telegram 全局约 30 msg/s、单 chat 1 msg/s，群发要排队。
- **并发模型**：多会话并行时，做到"会话内串行、会话间并行"，否则上下文会交错污染。

## 可复用建议

- Adapter 保持"dumb"，所有判断进 router/core，业务逻辑永远只有一份；
- envelope 带版本号和平台能力位（如 `supports_buttons`），功能降级写在 renderer；
- 本地开发挂一个 echo adapter，假装是第三个平台，测试不碰真实 API；
- 日志始终带 `platform` 字段，跨端问题一查便知。

## 总结

这套结构的本质：把平台差异压进两端薄薄的 adapter/renderer，中间的 agent 与路由完全中性。之后要接 Slack 或飞书，只是再加一对 adapter/renderer 的事。代码量不大，但边界划清之后，双端的维护成本基本等于单端。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/3f92d315ee54c8a3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/3634ed5ba2c873e1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/616785601761e468.png)

