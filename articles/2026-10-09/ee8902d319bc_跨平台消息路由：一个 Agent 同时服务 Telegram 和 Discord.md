---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 40949
source: 综合讨论
publishedAt: 2026-10-09
---

## 背景

我们组原来有一个跑在 Telegram 群里的 OpenClaw Agent，做值班提醒和文档问答。后来一部分协作迁到了 Discord，第一反应是照着再复制一个 bot。跑了两周就暴露问题：两边记忆不互通、提示词改一处漏一处、排障要在两套日志里来回翻。于是决定收拢成一套结构：**一个 Agent 核心，前面挂统一路由层，Telegram 和 Discord 都只是接入端**。

## 问题

跨平台接入不是"多写一个 listener"，真正的成本在三类差异：

- **协议差异**：消息结构、mention 语法、附件、回复/线程语义都不同；
- **限制差异**：Discord 单条 2000 字符，Telegram 单条 4096，限频策略各自一套；
- **格式差异**：Markdown 方言互不兼容，同一段回复原样转发必然翻车。

## 做法

架构上拆成四层，Agent 核心保持不动：

1. **统一消息信封**。定义内部 schema，两个 adapter 各自负责把原始事件归一化成信封，不做任何业务逻辑：

```json
{
  "platform": "telegram",
  "chat_id": "-100xxx",
  "user_id": "88xxx",
  "msg_id": "42",
  "text": "帮我查上周的值班表",
  "attachments": [],
  "reply_to": null
}
```

2. **入站路由**。以 `platform + chat_id` 为会话键投递队列，按 `msg_id` 去重——Telegram 的 long polling 重试和 Discord 断线 resume 都会造成重复投递。
3. **Agent 核心**。不感知平台，只消费信封、调 MCP 工具、产出结构化回复。
4. **出站渲染**。按平台渲染：Discord 走 embed + 分段，Telegram 用 HTML parse mode（避开 MarkdownV2 的转义地狱）。

## 踩坑点

- **Markdown 方言**是最先翻车的地方。最后放弃"一份文本到处发"，改为 Agent 输出中性结构（段落 + 代码块 + 列表），渲染层各发各的。
- **长回复分段**要按段落边界切。硬切 2000 字符会把代码块拦腰斩断，Discord 代码块不闭合会直接吞掉后半段。
- **限频**必须做独立出站队列：Telegram 单聊天约 1 msg/s、全局 30 msg/s；Discord 遇 429 要退避。按 `chat_id` 排队是最省事的方案。
- **长任务体验**：工具调用超过 10 秒用户就以为挂了。Telegram 用 `sendChatAction`，Discord 用延迟回复，成本很低但体感差别巨大。
- **身份不要急着合并**。同一个人在两个平台先当两个用户处理，等有真实需求再做绑定，否则排障要多背一层映射的心智负担。

## 可复用建议

- Adapter 保持薄：只做归一化和 I/O，业务逻辑全部沉到核心；
- 所有日志带 `platform + chat_id + msg_id` 三个字段，跨平台排障基本靠它们；
- 每个平台独立 feature flag，出问题可以单独熔断一条通道；
- 未来接 Slack、飞书这类平台，理论上只需要一个 adapter + 一个 renderer。

## 总结

这套结构的核心收益是"核心不重复"。路由层大约 500 行代码，换来的是一份记忆、一份提示词、一份日志；后续接新平台的边际成本，明显低于再养一个完整 bot。如果你的 Agent 目前只在单一平台跑，建议尽早把信封 schema 和渲染层抽象出来——等双平台并行之后再重构，迁移成本会高不少。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/8724342362b1582c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/163b7bfe7df13994.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/508593cba5dfacac.png)

