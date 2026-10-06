---
title: 跨平台消息路由实战：一个 Agent 同时服务 Telegram 和 Discord
feedId: 40656
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

我们的 Agent 最初只挂在 Telegram 上，服务一个内部群。后来社区主阵地迁到 Discord，第一个念头是再起一个实例——但提示词、MCP 工具配置、记忆目录全要复制一份，两边状态还会漂移。更干净的做法是：一个 Agent 内核，同时接入两个平台，把平台差异隔离在最外层。

## 问题

难点不在"连上两个平台"，而在三件事：

1. 入站消息格式不同：Markdown 方言、附件、回复引用、话题/线程语义都不一样；
2. 出站限制不同：长度上限、转义规则、rate limit 各有各的脾气；
3. 会话归属：同一个人在两个平台都发消息，算一个会话还是两个？

## 做法

**第一步：定义统一消息模型（canonical message）。** 所有入站消息先被适配器翻译成同一个结构：`channel / chat_id / user_id / text / attachments / reply_to / message_id`。Agent 内核只认识这个结构，永远不感知自己在哪个平台。

**第二步：适配器层只做三件事**——入站翻译、出站渲染、断线重连。Telegram 走 long polling（内网部署省掉公网 webhook），Discord 走 gateway WebSocket。出站时 Telegram 用 HTML parse mode，Discord 用普通 markdown，长度分别按 4096 / 2000 切段。

**第三步：会话路由键。** 我们的规则：群聊按 `channel + chat_id` 隔离，私聊按统一用户身份聚合（一张简单的 identity map 表绑定双平台账号），群聊之间互不串台。这个决定没有标准答案，但必须显式写下来——上下文串台是最难排查的 bug。

**第四步：速率与排队。** 每个 channel 一个独立令牌桶，Agent 产出先进全局队列再分发。Discord 的 per-channel 限流比 Telegram 严格得多，不能拿同一套节奏打两边。

配置层面，OpenClaw 的 `channels` 部分声明两个 adapter，`agent` 部分只有一份：模型、系统提示词、MCP 工具、记忆全部复用。上下线某个平台只动 adapter 开关。

## 踩坑点

- **Telegram MarkdownV2 转义**：`*_[]()~\`` 等一票字符都要转义，漏一个整条消息报 400。后来换成 HTML parse mode，只需处理 `<`、`>`、`&`，代价小得多。
- **切段切断代码块**：按字数硬切会把 ``` fence 切开，对端渲染直接乱掉。切段逻辑必须感知 fenced block，优先在段落边界切。
- **附件链路**：Telegram 的文件下载要带 bot token，Discord CDN 链接有时效。别把原始 URL 直接丢给 MCP 工具去抓，适配器应在入站时就落盘或换成带鉴权的临时链接。
- **重连后的重复投递**：gateway 和 polling 重连都会补推消息，入站和出站各做一层 `message_id` 去重。
- **Discord webhook 身份**：用 webhook 回复会显示成另一个"人"，社区里反复被问"这俩 bot 什么关系"。能走 bot 消息就别走 webhook。

## 可复用建议

1. **统一消息模型是杠杆最高的抽象**。做完之后接第三个平台（Slack、飞书）就是再写一个 adapter，内核一行不动。
2. **格式化决策全部留在边缘**，Agent 输出统一为普通 markdown + 结构化块，不在提示词里写平台特有语法。
3. **日志强制带 `channel + message_id`**，跨平台问题基本靠这两个字段定位。
4. **上线前跑 dry-run**：把 canonical JSON 发到测试频道，肉眼审一遍再放真实流量。
5. **会话键先写成文档再动手**，它是产品决策，不是实现细节。

## 总结

跨平台路由的本质是"一个内核，多个边缘"：平台差异全部消化在适配器层，Agent 对渠道保持无知；会话归属这类问题要在写代码前想清楚。这套结构在我们环境稳定跑了两个多月，双平台日均几百条消息，没再因为平台差异改过内核。如果你想接 Slack 或飞书，欢迎按这个模板贡献 adapter。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/2abf7345bd0f92a8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/a9fa9dfc7bca60ba.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/9a274011c8a592ff.png)

