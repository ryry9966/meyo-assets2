---
title: 跨平台消息路由：让一个 OpenClaw Agent 同时服务 Telegram 和 Discord
feedId: 43000
source: 综合讨论
publishedAt: 2026-10-30
---

## 背景

一开始我在 Telegram 上跑了一个 OpenClaw 实例，纯个人用途：提醒、查资料、跑点自动化。后来组内同事也想用，但他们习惯在 Discord 上沟通。摆在面前的选项有两个：再部署一个实例，或者让同一个 Agent 同时接两个渠道。

我选了后者，原因很实际：第二个实例意味着配置、密钥、记忆、定时任务全部翻倍，而且两边的上下文完全割裂——在 Telegram 里聊到一半的事，换到 Discord 就"失忆"了。OpenClaw 的 channels 设计本身就支持多渠道接入，真正的难点不在"接上"，而在路由。

## 问题

把它拆开，其实是三个子问题：

1. **会话隔离**：Discord 群里的对话不能污染 Telegram 私聊的上下文，反之亦然。
2. **回复路由**：Agent 主动触发的消息（定时提醒、任务结果）必须回到正确的渠道和会话。
3. **平台差异**：Markdown 方言、mention 语义、限流策略，两边都不一样。

## 做法

1. **多渠道接入**：配置里同时启用 telegram 和 discord 两个 channel，各自填 bot token。Telegram 用 polling 起步，Discord 走 gateway 连接。
2. **会话键设计**：session key 用 `channel:chatId:userId` 三元组，私聊和群聊天然隔离，跨渠道上下文不会串。
3. **回程路由**：所有入站消息在适配层记下来源三元组，出站消息一律按这个绑定回发。定时任务不依赖默认值，显式指定目标渠道。
4. **格式归一化**：Agent 内部统一输出标准 Markdown，各渠道适配器负责转成平台方言，转义逻辑收敛在一处维护。
5. **出站队列**：加一层带速率控制的队列（Telegram 每聊天约 1 msg/s，Discord 每频道 5 msg/5s），失败指数退避重试。

## 踩坑点

- Discord 开发者后台没开 **Message Content Intent**，bot 收到的消息内容全是空的，白白排查了一小时。
- 定时任务默认发到"第一个注册的渠道"，一次私人提醒被发进了团队频道。教训：主动消息必须显式绑定目标，不存在合理的默认值。
- Telegram 的 MarkdownV2 转义：没转义的下划线和括号会直接导致发送失败，而且报错信息很隐晦。
- 早期图省事用了全局 session，结果 Discord 侧能问出 Telegram 里的对话内容，属于隐私事故，幸好只发生在测试期。

## 可复用建议

- 把渠道当适配器写，Agent 核心不感知平台。以后加 Slack、WhatsApp，只是多写一个 adapter。
- 日志统一带 `channel:` 前缀，排障时按前缀 grep，两条链路一眼分开。
- 灰度顺序：先用测试账号在两个平台各跑通私聊，再放开群聊和 mention 触发，最后上队列和路由表。
- 任务到渠道的映射写成配置，不要硬编码。

## 总结

一个 Agent 服务多平台，在 OpenClaw 的 channel 模型下完全可行，工程量集中在路由、会话隔离和格式归一化三件事上。节奏上建议先单平台跑稳，再开第二个渠道，最后补队列和显式路由。连接本身一天就能搞定，真正值得花时间的，是那些不显眼但会出事的细节。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/e7a5f2ea10bbff16.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/97fcab2d4937cdf5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/faeec2fe3cb5b634.png)

