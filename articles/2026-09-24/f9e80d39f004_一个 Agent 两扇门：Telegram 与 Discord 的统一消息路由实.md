---
title: 一个 Agent 两扇门：Telegram 与 Discord 的统一消息路由实践
feedId: 38665
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

我们此前在 Telegram 和 Discord 各跑一个 bot，同一套 Agent 逻辑复制了两份：prompt 改一次要同步两处，MCP 工具配置各自漂移，排障时两边行为经常对不上。这次重构的目标很朴素：一个 Agent 实例做大脑，两个平台只是不同的"门"。

## 问题

实际工作量不在 Agent，而在边界上：

1. **消息模型不一致**：Discord 有 guild/channel/thread 和 snowflake，Telegram 是 chat_id/topic，回复、附件、线程语义都对不齐；
2. **会话归属**：同一个人在两个平台，算一个用户还是两个？
3. **出站格式**：Discord 2000 字符限制加自家 markdown 方言，Telegram 4096 字符加 MarkdownV2 的转义地狱；
4. **传输层差异**：Discord 网关要维持 websocket 心跳，Telegram webhook 必须快速回 200，而 LLM 调用动辄十几秒。

## 做法

架构分三层：adapter → 归一化/路由 → Agent。

1. **定义内部归一化消息 schema**：`platform`、`chat_id`、`user_id`、`reply_to`、`attachments`、`thread` 等字段。这是全系统的契约，单独版本化。
2. **两个 adapter 只做翻译**：入站把平台消息转成 schema，出站把 Agent 输出转回平台格式。adapter 是插件，不写业务逻辑。
3. **路由用配置表**：

```yaml
routes:
  - platform: telegram
    chat_id: "-1001234567"
    workspace: support-cn
  - platform: discord
    channel_id: "1122334455"
    workspace: support-cn
```

   默认路由加少量覆盖，够用，别上来就搞复杂 DSL。
4. **会话 key 用 `(platform, chat_id, user_id)`**。我们最终决定不做跨平台身份合并——两个会话独立维护，避免误关联。真有需求再加一张映射表，但先别。
5. **adapter 与 Agent 之间放异步队列**：Telegram webhook 先回 200 再慢慢处理，Discord 断线重连用指数退避 + resume。
6. **出站统一走 shaping 层**：按平台限制分片、转换 markdown 方言、过 token bucket 限速。

## 踩坑点

- Telegram MarkdownV2 要转义十几个字符，别硬扛，直接用 HTML parse mode；Discord 用原生 markdown。两套输出最省事。
- 分片切断代码块是高频事故，chunker 必须追踪 fence 状态，闭合再重开。
- Discord 断线 resume 后会补发消息，入站必须按 `(platform, message_id)` 去重，否则 LLM 白跑还重复回复。
- 想做"流式编辑"时注意：Discord 每频道编辑限频很死，改成 typing 状态 + 最终一次性发送更稳。
- Discord slash command 3 秒内必须响应，先 ack 再 followUp。
- Discord CDN 的附件链接有时效，Agent 后续要用文件就得及时下载落盘。

## 可复用建议

- adapter 保持"笨"，Agent 保持对平台无感知，中间的 schema 是唯一契约；
- 加一个 dry-run 模式：只打印归一化后的消息不真正回复，联调能省一半时间；
- 日志贯穿一个 correlation id，从入站 adapter → Agent → 出站能串起来查；
- 出站限速按平台分桶，别共用一个队列，一个平台被限流不该拖死另一个。

## 总结

一个 Agent 服务两个平台，在中小规模下完全可行。工程量的 80% 花在归一化边界和出站 shaping 上，而不是 Agent 本身。先把 schema 定稳，以后接入第三个平台，本质上只是多写一个 adapter 的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/77dd96befaf63c33.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/9917252a8ed56894.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/b125667f8bf605c2.png)

