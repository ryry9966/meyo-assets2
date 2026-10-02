---
title: 跨平台消息路由实战：让一个 Agent 同时守住 Telegram 和 Discord
feedId: 40129
source: 综合讨论
publishedAt: 2026-10-02
---

## 背景

我们社区的用户一半在 Telegram、一半在 Discord。最初偷懒，两个平台各挂一个独立 bot，各自跑一份 agent。跑了一个月问题就出来了：人设漂移、长期记忆不同步、提示词改了一端忘另一端，排障还得看两套日志。后来收敛成**一个 Agent 同时服务两端**，这篇记录做法和踩过的坑。

## 问题

表面上是"多接一个通道"，实际要解决三件事：

1. **会话与身份路由**：同一个人在两个平台，算一个用户还是两个？上下文会不会串台？
2. **消息格式差异**：Discord 的 Markdown 和 Telegram 的 MarkdownV2/HTML 互不兼容，LLM 原样输出很容易 400。
3. **回复语义与限速不同**：Discord 有线程和斜杠命令，Telegram 有话题；两端限速策略也完全不一样。

## 做法

**1. 单实例 + 双通道薄适配器。** Agent 核心只跑一份，gateway 配置里声明两个 channel，各挂一个轻量 adapter（示意）：

```yaml
channels:
  telegram:
    botToken: xxx
    mode: polling      # 家庭宽带省事，不上 webhook
  discord:
    botToken: yyy
    intents: [guild_messages, message_content]
routing:
  sessionKey: "{channel}:{chat_id}"
```

**2. Session key 显式带 channel 前缀。** `telegram:12345` 和 `discord:98765` 天然隔离，避免上下文串台。需要跨平台的长期画像走独立的 memory 层，不要靠混 session 实现。

**3. 内部统一消息信封。** Adapter 只负责把平台消息归一化：文本、附件、回复引用、mention。出站默认"保守纯文本 + 平台渲染器"两级，先降级再按目标端二次渲染。

**4. 触发规则差异化。** Telegram 群里 @bot 或回复触发，Discord 用 mention + 斜杠命令；私聊两端都默认直接回答。

## 踩坑点

- **Markdown 方言**：把 Discord 风格输出直接发 Telegram 会报错。格式兼容不要指望 LLM 自觉，出站统一走渲染器。
- **回复循环**：两端 bot 如果通过桥接进了同一个群，会互相当成用户聊起来。必须过滤自消息，并对已知 bot id 静默。
- **Discord 的 Message Content Intent** 不开时收不到正文，表现为"只收到空消息"，极易误判成模型问题，先查这里。
- **限速**：Telegram 每 chat 约 1 msg/s；Discord 每 channel 5 req/5s，编辑消息更快触发。长回答别流式刷编辑，聚合后一次发。
- **webhook 需要公网 HTTPS**，没有固定出口就用 polling，别硬上。

## 可复用建议

- **Adapter 保持薄**：只做鉴权、格式、限速，不做业务判断；路由策略全部收敛到配置文件。
- **Session key 设计放在最前面定**，后期迁移很痛。
- 日志和指标统一带 channel 标签，排障时一眼分辨问题在哪端。
- 两个通道独立健康检查，一端 token 失效不影响另一端运行。

## 总结

一句话：**胖核心、薄适配器、显式会话路由**。收敛之后提示词只维护一份，记忆一致，双端体验稳定。成本主要集中在前两周磨平格式和限速细节，之后基本是无感维护。如果你的用户也分散在多个 IM，建议尽早走这条路，越晚迁移会话数据的包袱越重。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/4a7c2fbddbacfac7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/9aed5fd1a1f1af02.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/dfc3c045c772fd71.png)

