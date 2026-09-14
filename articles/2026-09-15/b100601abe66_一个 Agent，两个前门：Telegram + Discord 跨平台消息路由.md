---
title: 一个 Agent，两个前门：Telegram + Discord 跨平台消息路由实践
feedId: 37618
source: 综合讨论
publishedAt: 2026-09-15
---

我们社区的用户一半在 Telegram，一半在 Discord。这篇记录一下怎么用一个 Agent 实例同时服务两边，以及过程中踩到的坑。

## 背景

早期为了省事，两边各跑了一个 bot，各自接模型、各自的脚本。后果很直接：两份 prompt 要同步维护，记忆不互通，行为慢慢漂移——同一个问题在两边能得到不同答案。后来的思路收敛成一句话：**Agent 只留一份，平台降级为通道插件，差异全部在路由层消化。**

## 问题

拆开看核心是三件事：

1. **会话隔离**：同一个来源的上下文要独立，但长期记忆希望共享；
2. **格式差异**：Discord Markdown 和 Telegram MarkdownV2 的转义规则完全不同，长度上限也不同（2000 vs 4096）；
3. **消息形态**：图片、附件、回复引用、线程（Discord thread / Telegram 话题）在两边语义不一致。

## 做法

在 OpenClaw 网关侧大致这样组织：

- 一个 agent workspace，一份系统提示词，MCP 工具挂同一套；
- 通道层启用 telegram 和 discord 两个插件，各自配 token；
- 路由规则给每个来源分配 session key，用 `telegram:{chat_id}` / `discord:{channel_id}` 这样的格式，保证会话隔离；需要共享的长期知识放 workspace 的记忆文件里——**会话隔离、知识共享**；
- 出站统一走一个 formatter：按平台切分长度、处理转义；
- 消息内部统一成 envelope：`{channel, sender, chat_id, text, media[], reply_to}`。Agent 逻辑只认 envelope，不感知平台。

## 踩坑点

- Telegram 用 long polling 时误起了第二个实例，两边抢 update 报 409；确认单实例后切 webhook 才稳定。
- Discord 不开 MESSAGE CONTENT intent，群里收到的消息 content 是空的，排查了半天以为是正则问题。
- MarkdownV2 转义极其挑剔，下划线、句点都会炸。最后放弃在 agent 侧做富文本，输出纯文本 + 链接，格式化全部放在边缘适配层。
- Discord 频道限速约 5 条/5 秒，批量推送要做队列；Telegram 单聊约 1 条/秒，量级完全不同。
- 群聊 session key 一开始用了 user_id，两个人私聊上下文直接串了；改成 chat 维度才对。
- 触发逻辑不一致：Discord 靠 @mention，Telegram 靠命令前缀，统一放到路由层判断，agent 不掺和。

## 可复用建议

- **适配层做薄**：通道插件只做收发、格式化、限速，业务逻辑全在 agent；
- **envelope schema 先定死**，后面接 Slack、Matrix 就是复制一个插件的事；
- 日志强制带 channel 标签，跨平台问题能一条 trace 拉通；
- 每个通道加独立心跳检查（定时发消息到测试频道）——bot 静默挂掉是最难发现的故障。

## 总结

这次改造最大的收益不是省了一个进程，而是把“平台差异”压缩到了边缘：agent 不再关心消息从哪来，路由层负责翻译。之后再接新平台，工作量是写一个适配器，而不是再养一个大脑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/2b8cdaff7cb5b3cb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/c2460e1a77892775.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/28252c5ff538bc9d.png)

