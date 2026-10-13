---
title: 跨平台消息路由实战：一个 Agent 同时服务 Telegram 和 Discord
feedId: 41393
source: 综合讨论
publishedAt: 2026-10-13
---

## 背景

我们的社群用户一半在 Telegram、一半在 Discord。最初是两个独立 bot，各自挂一套 prompt 和工具，结果配置漂移、记忆不同步，改一个 bug 要发两版。这次重构的目标很朴素：**Agent 核心只有一份，平台差异全部收敛到适配层**。

## 核心问题

真正麻烦的不是"多接两个 SDK"，而是三件事：

1. **会话身份**：同一个人在两个平台的 ID 完全无关，跨平台记忆怎么挂？
2. **消息形态差异**：Discord 单条 2000 字符、支持 embed；Telegram 4096 字符，MarkdownV2 转义规则苛刻。
3. **事件语义不同**：Discord 有 thread 和 reaction，Telegram 有 reply_to 和 topic，回复目标不能搞混。

## 做法

整体是经典三层结构：

```
Telegram Adapter ─┐
                  ├─→ 统一消息总线 → Agent Core → 渲染器 → 各自 Adapter 回复
Discord Adapter ──┘
```

**第一步：定义统一信封。** 所有入站消息先归一化：

```json
{
  "platform": "telegram | discord",
  "chat_id": "...",
  "thread_id": null,
  "user_id": "...",
  "msg_id": "...",
  "text": "...",
  "attachments": []
}
```

Agent Core 只认这个结构，不知道底下是什么平台。

**第二步：会话键设计。** 我们用 `platform:chat_id:thread_id` 做会话主键，而不是 user_id——同一用户在群聊和私聊本该是两个上下文。跨平台身份合并做成显式 mapping 表，用户主动绑定后才共享长期记忆，默认不合并，避免串台。

**第三步：出站渲染器。** Agent 输出统一的中间格式（纯文本 + 结构化块），渲染器负责平台化：

- Discord：按段落切到 2000 字符内，代码块不跨条切断；超长内容转附件文件。
- Telegram：MarkdownV2 转义全部收在渲染器里，Core 永远不产出转义后的字符串。
- 两个平台各自挂令牌桶限速，Discord 的 per-channel 限速比全局更容易触发，要分开配。

## 踩坑点

- **Discord 的 Message Content Intent 默认关闭**，Developer Portal 不手动打开，bot 只能收到空消息，第一晚就卡在这。
- **Telegram 轮询 vs Webhook**：内网部署没公网证书就老实用 long polling，别硬上 webhook；Discord 只有 Gateway WS 一条路，重连后事件可能重放，必须用 `msg_id` 做幂等去重。
- **长任务阻塞**：工具调用超过 3 秒就先发"思考中"占位（Discord 用 typing 状态，Telegram 用 sendChatAction），否则用户会重复发消息触发并发。
- **Markdown 转义**：MarkdownV2 需要转义 `_ * [ ] ( ) ~` 等近 20 个字符，漏一个整条消息直接 400。教训是渲染器里写白名单式转义函数，配单测覆盖。

## 可复用建议

1. **适配层无状态**：会话、记忆、工具全在 Core，Adapter 只做收发和格式转换。以后加 Slack 就是新增一个 Adapter，Core 零改动。
2. **信封带 version 字段**：结构演化时旧消息可灰度迁移。
3. **日志统一带 `platform + msg_id`**：排障时能从一条回复一路追到入站原始事件。
4. **用 fixture 回放做测试**：把两个平台的真实消息脱敏后存成 JSON，CI 里跑归一化和渲染的快照测试，比手工双端点验省太多时间。

## 总结

改造完成后，核心代码里没有一行 `if platform ==`。跨平台路由的难点从来不在"多接一个 SDK"，而在会话模型和消息语义的抽象是否干净。先把信封结构和会话键定对，剩下的平台差异就只是渲染层的体力活。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/2176f32caed3f4fe.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/828b03708a1d256e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/64bcc948ac25ba8a.png)

