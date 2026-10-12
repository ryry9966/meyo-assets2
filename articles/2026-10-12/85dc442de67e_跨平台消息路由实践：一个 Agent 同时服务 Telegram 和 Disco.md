---
title: 跨平台消息路由实践：一个 Agent 同时服务 Telegram 和 Discord
feedId: 41338
source: 综合讨论
publishedAt: 2026-10-12
---

## 背景

接 OpenClaw 的常见路径：先在 Telegram 挂个 bot，跑通「消息进来 → Agent 处理 → 回复出去」这条链路。跑稳之后，用 Discord 的同事就会来问：能不能把同一个 Agent 也接过去？需求听起来简单，但最常见的做法——再起一个实例、复制一份配置——通常一两周内就会出问题。

## 问题

- **两个实例等于两份记忆**：上下文、会话状态、工具调用结果完全不通，同一个 Agent 在两边各聊各的。
- **平台消息模型差异大**：Markdown 方言不同，长度限制不同（Telegram 4096 字符，Discord 2000），附件和语音的处理路径也不一样。
- **身份分裂**：同一个人在两个平台是两个 ID，想合并需要额外的绑定机制。

## 做法

核心思路一句话：**把渠道适配和 Agent 核心拆开，中间加一层规范化。**

1. **定义统一消息模型**。字段不用多：`channel`、`chat_id`、`sender`、`text`、`attachments`、`reply_to`。所有平台消息进来先翻译成这个结构。
2. **写两个薄适配器**。Telegram 走 Bot API 长轮询，Discord 走 Gateway WebSocket。适配器只做翻译和收发，不做业务判断；MCP 工具调用全部留在核心层，不感知平台。
3. **会话路由**。路由 key 取 `(channel, chat_id)`，映射到 Agent 的 session。同一会话的消息串行进 Agent，避免并发写坏上下文。
4. **出口再翻译**。Agent 回复是普通 Markdown，适配器负责按平台转义、切分（按段落切，别按字数硬切）、超长降级成文件。
5. **附件走本地临时目录**。两边下载落盘、Agent 处理、回复时重新上传，不透传原始 URL。

## 踩坑点

- MarkdownV2 转义是重灾区。LLM 输出的代码块里全是特殊字符，直接发 Telegram 就是 400。实践用 HTML parse mode 加手动 escape 兜底。
- Telegram 长轮询不能双实例跑，会 409 冲突；Discord 记得开 Message Content Intent，否则收到的消息内容全是空的——这个坑不报错，最难查。
- Discord Gateway 偶尔重发事件，入口要按 message id 做幂等去重。
- 异步混用要小心：Discord 库是 asyncio 的，Telegram 轮询和 LLM 调用如果同步阻塞，整个事件循环卡住，两边一起没响应。
- Telegram 语音是 ogg/opus，Discord 附件 URL 会过期，都要尽快落盘。

## 可复用建议

- 适配器保持「笨」，业务全在核心层。以后接 Slack、QQ 就是再加一个适配器的事。
- 先写一个 CLI 假适配器做本地调试，不用每次都去平台上发消息，能省大量时间。
- 日志统一加 channel 前缀，出问题时一眼定位是哪条链路。
- 会话加 TTL；群里 @ 消息和私聊用不同的响应策略。

## 总结

这件事的本质不是「接两个平台」，而是逼着把 Agent 核心做成平台无关。规范化层做扎实之后，多渠道从开发问题退化成配置问题。这套结构稳定跑了一个多月，两边上下文一致、状态唯一，日常维护成本基本只剩适配器本身。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/23e6f025870e5097.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/ca903660356f0608.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/9112537dc5cc3f15.png)

