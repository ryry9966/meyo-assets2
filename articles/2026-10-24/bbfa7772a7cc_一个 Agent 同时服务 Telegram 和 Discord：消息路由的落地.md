---
title: 一个 Agent 同时服务 Telegram 和 Discord：消息路由的落地记录
feedId: 42465
source: 综合讨论
publishedAt: 2026-10-24
---

## 背景

我们社区的用户分布在两个平台：一部分人习惯 Telegram，海外贡献者集中在 Discord。之前是两个 bot、两份 prompt、两套记忆，同一个问题经常被回答两次且答案不一致，维护成本翻倍。上个月我们把两条链路合并到了同一个 OpenClaw Agent 上，核心思路是"适配层归平台，智能层唯一"。这篇记录一下做法和踩过的坑。

## 问题

直接把两个 bot 各自接到 Agent，会遇到三类麻烦：

1. **消息结构不同**。Telegram 的 reply、编辑、markdown 转义规则，和 Discord 的 thread、embed、2000 字限制，完全是两套语义。
2. **会话割裂**。同一用户在两个平台身份不同，session key 若直接用平台原始 ID，上下文无法复用；若强行合并，又有串台的隐私风险。
3. **投递行为不同**。Discord gateway 断线重连、Telegram webhook 重复推送，都会造成重复投递或丢消息。

## 做法

整体分四步：

**第一步：定义统一信封。** 所有入站消息先归一化：

```yaml
envelope:
  trace_id: uuid
  platform: telegram | discord
  chat_key: "tg:chat:-100123" | "dc:chan:9876"
  user_ref: 平台内用户标识
  text: 归一化纯文本
  reply_to: 可空
  attachments: [url, mime]
```

关键是 `chat_key`：会话按"平台+频道"粒度隔离，默认不跨平台合并记忆。

**第二步：写两个薄适配器。** Telegram 走 webhook，Discord 走 gateway bot。适配器只做三件事：拉消息、转信封、把出站响应渲染回平台格式，不含任何业务逻辑。

**第三步：中间加队列。** 适配器生产、Agent 消费，Redis Stream 就够。入站按 `chat_key` 分区保证同一会话顺序，出站按平台限速策略消费；重复投递靠 `trace_id` 幂等去重，TTL 10 分钟。

**第四步：出站渲染按平台分流。** 同一份 Agent 回复，Telegram 走 HTML parse mode（原因见踩坑），Discord 优先 embed，超长按代码块边界切段。

## 踩坑点

- **Telegram MarkdownV2 转义是无底洞**。用户消息里的下划线、星号、反引号都会让推送 400。改用 HTML parse mode 并对用户内容做一次 escape 后，问题少了 90%。
- **切分消息别按字符数硬切**。Discord 2000 限制按 code point 算，代码块切一半就废了。我们改成优先在代码块围栏处切，其次空行。
- **webhook 重试导致重复回复**。Telegram 超时后会重推同一条 update，适配器必须先落幂等键再处理，否则用户收到两遍答案。
- **回复映射别偷懒**。Telegram 的 reply_to 在 Discord 对应 message reference 和 thread，直接丢掉会让多轮追问的上下文观感很差。
- **typing 指示**。Agent 推理超过 20 秒时，两个平台都要及时发 typing，否则用户以为 bot 挂了。

## 可复用建议

1. **适配器保持"哑"**。平台差异全部消化在适配器层，Agent 和 skill / MCP 工具永远只面对统一信封。以后接 Slack 只是再加一个适配器。
2. **默认会话隔离，显式合并**。跨平台身份打通（如绑定命令把两个 ID 关联到同一 profile）做成用户主动操作，不要自动猜测。
3. **trace_id 贯穿全链路**。日志里用 `tg:xxx` / `dc:xxx` 前缀，出问题能从一条回复反查到原始消息。
4. **先灰度**。新平台先接一个测试频道跑一周，重点看限速日志和转义报错。

## 总结

跨平台不是"多接一个 bot"的问题，而是语义归一化、会话策略、投递可靠性三件事叠加的工程问题。适配层做薄、队列做幂等、渲染按平台分流——这三点立住了，一个 Agent 服务 N 个平台基本就是复制适配器的工作量。目前这套路由跑了三周，日均几百条消息，重复回答和双份 prompt 维护的问题已经彻底消失。如果有人在接第四个平台，欢迎在楼下同步适配器细节。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/39eaa9adb274018c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/826487dcde227934.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/997fd23907faeae7.png)

