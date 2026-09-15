---
title: 跨平台消息路由：让一个 Agent 同时服务 Telegram 和 Discord
feedId: 37755
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

社区用户分散在 Telegram 和 Discord 两个平台。最初的方案是跑两个 Agent 实例，各挂一个 Bot token。短期内能用，但问题很快出现：两边的提示词版本漂移、记忆不共享、改一处配置要人工同步两处。这周把架构收敛成「一个 Agent 核心 + 两个薄适配器」，记录一下过程。

## 问题

本质上是三类问题：

1. **传输层不同**：Telegram 走 Bot API（长轮询或 webhook），Discord 走 Gateway WebSocket，事件模型完全不同；
2. **消息格式不同**：Markdown 方言、长度上限（Telegram 4096 / Discord 2000）、转义规则各不相同；
3. **状态归属**：逻辑会话用什么 key？跨平台身份要不要打通？

## 做法

核心原则：适配器只做协议翻译，所有业务逻辑收进 Agent 核心，工具统一挂在 MCP server 上，保证平台无关。

1. **定义统一信封**。入站消息归一化为 `{platform, chat_id, user_ref, text, attachments, reply_to}`；出站是规范化结构，渲染交还给适配器。
2. **写两个薄适配器**。Telegram 用长轮询（内网部署没有公网 HTTPS，省掉 webhook），Discord 用 Gateway，记得在开发者后台打开 Message Content Intent，否则收不到消息正文。
3. **单点分发**。所有入站消息进同一个队列，按 `platform:chat_id` 做 session key，同一会话串行处理，避免两个平台同时触发导致并发写坏状态。
4. **出站渲染层单独抽出**：Markdown 转换、按平台上限分片、长输出转文件附件。Telegram 建议直接用 HTML parse mode。
5. **身份映射选配**。默认不打通跨平台身份（`tg:123` 和 `dc:456` 就是两个用户），有明确需求再引入人工绑定的映射表，不要自动猜测合并。

## 踩坑点

- Telegram 的 MarkdownV2 转义是灾难级体验：下划线、星号、方括号等十几个字符都要转义，LLM 输出几乎必炸。换 HTML mode 后基本解决。
- 分片会劈开代码块。按段落边界切，检测到未闭合的代码围栏就顺延到块结束再切。
- 两边都有频控：Discord 单频道约 5 条/5 秒，Telegram 单聊天 1 条/秒。出站必须走队列加退避重试，老老实实处理 429。
- Discord 断线要支持 resume；Telegram `getUpdates` 的 offset 要持久化，否则重启后会重复消费旧消息。
- Discord 的 mention 格式（尖括号包 user id）跨平台无意义，渲染层直接替换成用户名文本。

## 可复用建议

- 适配器保持「笨」：不解析业务语义，不做除重试策略之外的任何决策；
- 信封结构加版本号字段，后期改 schema 不至于静默坏掉；
- 每条日志都带 `platform + chat_id`，排障效率差一个数量级；
- 加一个 dry-run 通道：所有出站先镜像到测试频道，肉眼确认渲染效果再放开。

## 总结

跨平台路由的关键不是「多接一个 API」，而是把协议差异锁死在适配器里，让 Agent 核心面对一个稳定抽象。收敛之后，改提示词、加工具只动一处，两个平台的用户体验也保持一致。两个适配器加渲染层大概几百行代码，但前期把信封 schema 想清楚，比后面任何优化都重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/17ef1b28c99cb964.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/767524121ea618dc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/eca1f8afa2d240d2.png)

