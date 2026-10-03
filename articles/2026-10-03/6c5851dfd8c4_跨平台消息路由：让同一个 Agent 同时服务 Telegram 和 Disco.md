---
title: 跨平台消息路由：让同一个 Agent 同时服务 Telegram 和 Discord
feedId: 40246
source: 综合讨论
publishedAt: 2026-10-03
---

## 背景

我们社区基于 OpenClaw 维护了一个问答/值班 Agent，早期只挂在 Telegram。后来 Discord 这边的用户多起来，需求很自然：不要复制一套 Agent，让同一个实例同时服务两个平台。实践下来比预想的复杂，但结构想清楚之后，接第三个平台反而很快。

## 问题

难点不在"再接一个 bot"，而在两边的语义差异会击穿你原有的假设：

- 消息上限不同：Telegram 单条约 4096 字符，Discord 是 2000；
- 格式方言不同：Discord 的 mention 是 `<@id>`，Telegram 的 MarkdownV2 转义出了名地难缠；
- 会话语义不同：Discord 有 thread，Telegram 只有 reply/topic；
- 身份不同：同一个人在两边是两个独立的 user_id。

如果 Agent 核心里开始出现平台特判，两个月后就是 if-else 坟场。

## 做法

1. **统一信封**。定义规范化消息结构：platform、channel、thread_ref、user、text、attachments。两个 adapter 各自把平台原始事件翻译成信封，Agent 核心只认信封。
2. **会话键设计**。默认按 `platform:chat_id(:thread)` 隔离会话——同一个人在两个平台就是两个独立会话。确需身份合并，单独建映射表并在日志里显式标注，绝不静默合并。
3. **出口渲染层**。回复不直接发送，先过 renderer：按平台上限分段（在段落和代码块边界切，不硬切）；Markdown 转平台方言；mention、emoji 做映射。
4. **媒体管线**。附件统一落中转存储，出站时按平台 API 重新上传，而不是透传 URL。
5. **路由表配置化**。用 YAML 描述"哪个群/频道 → 哪个 workspace/技能集"，改路由不发版。
6. **幂等与去重**。重启后 Telegram 用 offset、Discord 用最后一条 message id 做游标，处理前查重。

## 踩坑点

- Discord 回 thread 忘带 message_reference，回复漂到主频道，被用户当成失忆；
- Telegram 的 edited message 触发重复处理：我们最终策略是 edit 只记日志，不重新喂 Agent；
- 长回复硬切导致代码块断成两截，渲染全崩——切分点必须感知 ``` 围栏；
- 两边限速策略不同，Discord 突发消息直接 429，改成 per-channel 令牌桶才稳；
- 曾尝试自动合并两边身份，出过串会话事故后回滚。结论：合并必须是显式操作。

## 可复用建议

- adapter 只做两件事：入站规范化、出站渲染，任何业务逻辑都不许进 adapter；
- Agent 的 prompt 里永远不出现平台字段，平台差异全部消化在渲染层；
- 日志按 platform 打标签，排障时先过滤平台再看会话；
- 路由表和限速参数全部进配置，不要硬编码。

## 总结

跨平台的核心不是"多接一个 API"，而是把平台差异限制在边界层。信封统一 + 会话键隔离 + 出口渲染，这套结构后来接第三个渠道只花了一个下午。先立规范，再接渠道，顺序别反。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/a72858aaa997cc45.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/eb3b869b5b14472c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/95ac1db8a29584d5.png)

