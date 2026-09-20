---
title: 一个 Agent 服务两端：Telegram + Discord 消息路由的落地实践
feedId: 38220
source: 综合讨论
publishedAt: 2026-09-20
---

## 背景

之前两个平台各挂了一个 bot：同一套 prompt 复制两份，记忆和工具调用各自独立。用户在 Telegram 里说过的事，到了 Discord 又得从头讲一遍，改 prompt 要改两个地方。后来干脆收敛：Agent 核心只保留一份，Telegram 和 Discord 退化成两个“传输适配器”，跑了一个多月，把经验整理出来。

## 问题

平台差异比想象中大，直接在业务逻辑里写 if/else 很快会失控：

- 消息上限不同：Telegram 单条 4096 字符，Discord 2000；
- Markdown 方言不同，转义规则天差地别；
- 回复模型不同：Discord 有 thread，Telegram 只有 `reply_to`；
- 频率限制、媒体处理、身份模型也各是一套。

核心矛盾是：**Agent 逻辑必须一份，平台脏活必须全部外置**。

## 做法

**1. 定义统一信封（envelope）**

所有入站消息先归一化成一个 canonical schema：`platform / chat_id / user_id / message_id / text / attachments / reply_to`。适配器只做“翻译”，不碰业务。

**2. 核心与平台解耦**

Agent 核心通过 MCP 暴露工具，出站也走统一信封，路由层根据 `platform` 字段选发送函数。核心代码里不允许出现任何平台 SDK 的 import。

**3. 会话键设计**

私聊用 `{platform}:{user_id}`，群聊用 `{platform}:{chat_id}:{user_id}`。不要用 username 做键，两边的 username 都可能变。

**4. 出站格式化**

核心统一输出纯 Markdown，由适配器转成 Telegram HTML（MarkdownV2 转义黑名单太长，不值得）或 Discord markdown，并按平台上限分块。

**5. 限速队列**

每个平台一个独立发送队列：Telegram 群聊按 1 msg/s 节流，Discord 按 per-route 限制加 jitter，队列满时排队而不是丢弃。

配置大致长这样：

```yaml
channels:
  telegram: { mode: polling }
  discord:  { intents: [guild_messages, dm_messages] }
router:
  session_key: "{platform}:{user_id}"
  chunk_limit: { telegram: 4000, discord: 1900 }
```

## 踩坑点

- **分块切坏代码块**：分块器切在 ``` 中间会把后续渲染搞烂，分块前先做围栏配对计数。
- **Discord 断线重连**：gateway 要指数退避；RESUME 失败后必须全量同步一轮，否则静默丢消息。
- **Telegram 重复消费**：`getUpdates` 的 offset 处理不好会重放，入站按 `message_id` 做去重窗口。
- **附件只传 URL 会翻车**：Telegram 服务端经常拉不动带签名参数的 Discord CDN 链接，稳妥做法是下载后重传。
- **Mention 归一化**：Discord 是 `<@id>`，Telegram 是 `@username` 或 `text_mention` entity，统一转成 `user_id` 再进核心。

## 可复用建议

- 适配器保持“哑”：单适配器超过 200 行，大概率是业务逻辑漏进去了。
- 日志统一带 `platform` 标签，排查时按平台过滤，一半问题直接消失。
- 新接第三个平台（比如 Slack）的成本应该收敛到三件事：信封翻译、发送函数、格式化规则。超过这个量级说明抽象漏了。

## 总结

这个方案最大的收益不是省一个 bot 进程，而是把平台差异压缩到薄薄一层适配器里：prompt、记忆、工具只有一份，行为一致、可单测。代价是要认真设计信封 schema，并把分块、转义这些脏活老老实实做完。做完之后，扩平台基本只剩体力活。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/d05c6a2eda57d66d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/4851b4af1c083145.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/5ca24084c0545e39.png)

