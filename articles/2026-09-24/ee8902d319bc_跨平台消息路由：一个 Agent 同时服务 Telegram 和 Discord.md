---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 38770
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

OpenClaw 的 channel 机制天生是解耦的：gateway 进程跑在本地，Telegram 和 Discord 只是两个适配器，Agent 本体感知不到平台差异。听起来接两个 token 就完事，但真让一个 Agent 同时守两边，路由问题立刻浮出水面。这里记录我跑了两个多月的一套拓扑和教训。

## 问题

实际撞上的有三类：

1. **会话归属**：同一个人在两个平台都有账号，上下文要不要共享？共享了怕互串，隔离了又像失忆。
2. **出站格式**：Telegram 的 MarkdownV2 和 Discord 的 markdown 方言互不兼容，长度上限也不同（4096 vs 2000）。
3. **稳定性**：群聊被 @ 之后连续回复，很容易撞限流，甚至和别的 bot 形成回声循环。

## 做法

我的拓扑：一个 gateway + 两个 channel + 一个 Agent。

1. **会话拓扑先行**。接 channel 之前先定隔离策略：我选“每平台独立滚动会话，会话键带平台前缀（`telegram:chatId` / `discord:channelId:threadId`），共享层只放长期记忆文件”。两边上下文不互串，但“这个人是谁”是共享的。
2. **统一入站模型**。在 channel 插件里把消息归一化：sender、chat、thread、媒体引用，平台细节全部消化在适配器内，不出边界。
3. **每平台一个 renderer**。出站消息由 renderer 全权负责转义、分块（按各自上限、在代码块边界切）、解析失败时降级为纯文本。Agent 永远只产出一中立的中间格式。
4. **身份配对表**。Agent 提供一个配对工具：两个平台各发一条指令绑定到同一 user_id，存 SQLite。之后跨平台问“接着上次说”，能从 memory 里找回同一个人的上下文。
5. **出站队列**。所有回复先进本地队列，按平台做令牌桶限流，收到 429 读 `retry_after` 退避。绝不并发轰炸。

## 踩坑

- **MarkdownV2 转义是黑洞**。第一版直接透传 Agent 原文，Telegram 一直 400 且报错信息很隐晦。改成 renderer 统一转义 + 失败降级，才稳。
- **Discord 的 threadId 没进会话键**，两个 forum 帖的上下文串了。把 thread 层级编进 session key 才干净。
- **Telegram 重连后会重推 update**，同一条消息答了两遍。按 message_id 做幂等去重解决。
- **群聊没做 mention 门控**，Agent 和另一个 bot 互相礼貌问候刷了半小时屏。现在群消息默认只响应 @。

## 可复用建议

- Agent 的 prompt 里**永远不要出现平台名**，差异全部吸收在适配器层，否则迁移第三平台时会很痛。
- 路由规则**写配置不写代码**：哪些群需要 @、静默时段、优先通道，都能热改。
- 每条消息打 correlation id，两个平台的日志能对上，排障效率差一个量级。
- 节奏上先只开 DM 跑一周，再放群聊，最后才考虑跨平台转接。

## 总结

跨平台路由的难点不在“接上”，而在归一化、会话拓扑和出站纪律这三件事。适配器薄、Agent 无知、renderer 严格，这套结构跑下来，两个平台共用一个大脑是稳的。下一步计划把语音转写（走 MCP 工具）也压进统一消息模型，让 Telegram 的 voice note 和 Discord 的附件走同一条媒体管线。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/94490c0c0a93e9ba.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/dd32a6fc6793c0b4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/6d6c8deb532b4aba.png)

