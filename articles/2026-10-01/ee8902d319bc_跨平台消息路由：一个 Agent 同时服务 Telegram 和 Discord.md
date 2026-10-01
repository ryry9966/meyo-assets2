---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 40003
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

最初只接了 Telegram，自己用很顺。后来在 Discord 上建了个小社群，问题来了：是再养一个 bot、再维护一套 prompt 和记忆，还是让同一个 Agent 同时守两个平台？

OpenClaw 的架构里，channel（通道）和 agent 本体是解耦的：gateway 负责收发，agent 核心只面对规范化后的消息。所以"一个 Agent 服务多平台"理论上是配置问题，实践上是工程细节问题。这篇记录我把两个平台同时接进一个 Agent 的过程。

## 问题

真正要解决的只有三件事：

1. 消息进得来、回得出去，且格式不崩；
2. 会话上下文按预期隔离（或共享）；
3. 同一个人的两个身份要不要合并。

其余都是平台怪癖。

## 做法

**1. 通道接入。** Telegram 走 BotFather 拿 token，长轮询即可；Discord 在开发者后台建应用拿 token，务必打开 Message Content Intent。两个通道在 `~/.openclaw/openclaw.json` 的 `channels` 段同时声明，由同一个 gateway 进程托管。

**2. 会话键设计。** 默认用 `(channel, chat_id)` 做会话键：Telegram 私聊一个 session，Discord 每个频道一个 session。上下文天然隔离，不会出现"早上在 Telegram 聊的内容，下午在 Discord 答非所问"。确认需要跨端连续性后，再显式把某用户的两边身份指到同一 session——这个决定应该是主动的，而不是默认发生的。

**3. 格式适配层。** Agent 输出统一用基础 Markdown（粗体、代码块、列表），出口做降级：Discord 端几乎原样发；Telegram 端用 HTML parse mode 或退纯文本。MarkdownV2 那套转义规则不值得逐字符实现，除非你真的需要。

**4. 长度与频率。** Discord 单条 2000 字符上限，超长回复按代码块和段落边界切分，不要按字数硬切——拦腰切断代码块很难看。Discord 对同频道发消息有速率限制，批量通知记得加节流。

**5. 媒体归一化。** 语音消息：Telegram 是 ogg/opus，Discord 是附件 URL，进入 agent 前统一转格式，再走同一条转录工具链。图片同理，统一转成内部附件结构。

## 踩坑点

- Discord Intent 忘开，症状是 bot 能收到消息事件但 content 恒为空，我排查了半天 webhook 才想起来；
- Telegram 群里 bot 默认隐私模式收不到普通消息，要么关掉 privacy mode，要么让消息以 /command 或回复形式触发；
- 用 bot 自己的账号测试收不到自己的消息，两端都要用真人小号验证；
- 双平台在线后，同一个人两边各 @ 一次，Agent 会各答一遍。目前没做去重，靠 session 隔离 + 用户自觉；真要解决，可以加"最近 N 分钟内同内容去重"。

## 可复用建议

- Agent 核心保持通道无关，所有平台怪癖收在 adapter 层；之后接 Slack 或其他桥接，就是复制这个模式；
- 给每个通道维护一份能力矩阵：支持什么 Markdown、长度上限、有无按钮/表情回应，格式化函数按矩阵降级；
- 日志加 channel 前缀（`[tg]` / `[dc]`），跨平台排障省一半时间。

## 总结

一个 Agent 服务多平台，核心工作量不在"智能"，而在归一化与路由：会话键、格式降级、媒体转换、限流。OpenClaw 把 channel 解耦这件事做对了，剩下的就是老老实实填平台坑。接完两个平台后最大的体会是：先想清楚"谁和谁共享上下文"，比调任何 prompt 都重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/5f4f10afcb21cfcb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/ea695dbbb11a45a4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/ef97720033f4c040.png)

