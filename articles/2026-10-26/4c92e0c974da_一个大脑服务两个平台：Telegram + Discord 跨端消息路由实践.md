---
title: 一个大脑服务两个平台：Telegram + Discord 跨端消息路由实践
feedId: 42622
source: 综合讨论
publishedAt: 2026-10-26
---

## 背景

我们的 Agent 最早只接了 Telegram，群里陆续有人问能不能加 Discord。需求很明确：同一个“大脑”——同一份 system prompt、同一套工具（走 MCP）、同一段记忆——同时服务两个平台，而不是 fork 出两套各自维护的实例。

## 问题

难点不在接 SDK，而在平台方言全堆在外围：

- 收发模型不同：Telegram long polling，Discord 是 Gateway WebSocket；
- 格式不同：Telegram 的 MarkdownV2/HTML 与 Discord markdown 互不兼容；
- 限制不同：正文长度 4096 vs 2000，限流策略各一套；
- 身份体系不同：user_id / chat_id 完全不在一个命名空间。

如果让 Agent 直接感知这些差异，prompt 和工具层会越来越脏。正确的切法是：平台差异全部关进 adapter，大脑保持平台无关。

## 做法

1. **定统一信封。** 所有入站消息归一化成一个 schema：`platform / chat_id / thread_id / user_id / text / attachments / reply_to`。adapter 只负责“收—归一—投递”和“渲染—发送”。
2. **两个 adapter。** Telegram 用 long polling（个人部署够用，不依赖公网回调）；Discord 用官方 gateway 连接。
3. **路由层。** session key 取 `(platform, chat_id, thread_id, user_id)`，保证不同群、频道、线程互不串上下文；入站队列按 message_id 去重，同一 session 内串行处理，避免并发写坏记忆。
4. **出站渲染。** Agent 输出统一 markdown，渲染器按目标平台做转义和裁剪，超长按段落切块。
5. **媒体不传 URL。** 两边文件链接都带签名和有效期，必须在 adapter 层取字节再重传。
6. **观测。** 按平台记录发送延迟与错误码，429 类限流错误单独打标。

## 踩坑点

- Telegram webhook 与 long polling 互斥，切换时 `deleteWebhook` 没清干净，会静默收不到消息；
- Discord 对编辑同一条消息限流 5 次/5 秒，流式输出必须节流，我们最后改成攒句再 edit；
- Telegram MarkdownV2 的转义是黑洞，直接换 HTML parse mode，省一半调试时间；
- Telegram 群里默认 privacy mode 只收命令和 @，要么关掉，要么引导用户 @ 机器人；
- Discord 后台不开 Message Content Intent，收到的 content 是空的，且报错很不直观；
- 两个平台限流是独立预算，别共用一个全局限流器，也别裸奔。

## 可复用建议

- adapter 只做翻译，不做业务；大脑侧永远不出现平台字段；
- 录制真实 payload 当 fixture 回放测试，比端到端手测快一个量级；
- 流式输出按平台加开关：Discord 保守，Telegram 可以激进；
- 每平台独立限流桶 + 发送队列 + 带退避的重试。

## 总结

跨平台不是“多接一个 SDK”，而是分层：adapter 吸收方言，路由层保证 session 隔离与排队，Agent 核心一行不改。这套结构跑稳之后，接入第三个平台（比如 Slack）的边际成本，就只是再写一个 adapter。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/50ad37e2b16417d8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/0a4d3aea656fc70c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/4c8abce660af44fe.png)

