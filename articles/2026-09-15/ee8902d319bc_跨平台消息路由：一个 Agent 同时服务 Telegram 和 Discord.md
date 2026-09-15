---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 37691
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

我之前的状态是典型的"双实例浪费"：Telegram 放私人和家庭事务，Discord 放几个项目群。两边各跑一个 OpenClaw 实例，结果可想而知——记忆不同步（在 Telegram 里交代过的偏好，Discord 那边一无所知）、MCP 工具配置维护两份、API 花销直接翻倍。

后来收敛成单实例双渠道，跑了一个多月，稳定。这篇记录路由层的关键设计，都是踩过坑之后的结论。

## 核心问题

一个 Agent 服务多个渠道，本质上是三个问题：

1. **会话隔离**：不同群聊、不同人、不同平台的上下文不能串。
2. **回复格式**：Discord 单条 2000 字符上限，Telegram 的 MarkdownV2 转义是出了名的坑。
3. **触发策略**：私聊直连、群组按需触发，两个平台的机制完全不同。

## 做法

### 1. 单实例挂双渠道

`openclaw.json` 的 `channels` 里同时配置 `telegram` 和 `discord`，Gateway 常驻一个进程。Telegram 用 BotFather 拿 token 走 long polling；Discord 建 Application 拿 bot token 走 Gateway 连接。不需要公网回调地址，家宽环境友好。

### 2. 会话键必须带渠道前缀

会话键设计成 `channel:chatId` 形式，例如 `telegram:12345` 和 `discord:98765`。这一步不能省：Discord 和 Telegram 的用户/群组 ID 是各自平台的雪花数，**数值上完全可能撞车**，裸用 ID 做键会出现上下文串台，而且极难排查。

个人助理场景如果想让两边共享记忆，不要直接共享会话，而是让工具层读写同一个存储（笔记、任务列表），会话本身保持隔离。

### 3. 工具层只挂一份

MCP server 统一挂在 Agent 侧而不是渠道侧。天气、日程、脚本执行这些工具，两个渠道天然共享，配置只维护一份。

### 4. 触发与准入

- Telegram：私聊直连；群组开 privacy mode，靠 @ 触发，配 allowlist。
- Discord：DM 直连；服务器频道里 mention 触发，`guildId` 白名单控制准入，避免被拉进陌生服务器后乱接消息。

### 5. 回复格式适配

在渠道出站前做一层薄适配：Discord 侧按 2000 字符切片，尽量在空行处断开，避免切开 code block；Telegram 侧默认用 HTML parse mode 替代 MarkdownV2，转义问题少一大半，报错类长文本直接降级 plain text。

## 踩坑点

- **Discord 收不到消息内容**：九成是 Developer Portal 里 Message Content Intent 没开，这个开关不在代码里，在后台。
- **长消息切片切坏代码块**：按字符数硬切会把 ``` 对半劈开，按"代码块整体不拆"的规则切。
- **双渠道同时回复触发限流**：Discord 有严格的 rate limit，出站加一个简单队列串行化即可，不用上消息中间件。
- **群组噪音**：Discord 频道不配 mention 触发的话，Agent 会试图回复每一条消息，当晚就被刷屏教学了。

## 可复用建议

- 渠道适配层做薄，只管收发和格式，业务逻辑全部下沉到 Agent 工具层。
- 会话键永远带渠道前缀，这条没有例外。
- 每个渠道单独记投递失败日志，排查时能立刻定位是哪个平台的问题。
- 上线前用 dry-run 模式跑一周路由规则，只打印不发送，验证触发逻辑。

## 总结

一个 Agent 服务 N 个渠道，工程上不复杂，复杂的是边界设计：会话怎么切、触发怎么控、格式怎么适配。把这三件事想清楚，加第三个、第四个渠道只是多写一份配置的事。有类似双平台需求的可以少走弯路。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/95e850b655c80565.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/db916de854d071ff.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/45a9a78c275b2686.png)

