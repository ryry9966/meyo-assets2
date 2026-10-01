---
title: 一套大脑，两个门口：让同一个 Agent 同时接住 Telegram 和 Discord
feedId: 39956
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

社区用户分散在 Telegram 群和 Discord 服务器两边，问题重复、知识割裂。与其维护两个各挂一套提示词和工具的 bot，不如让一个 Agent 核心同时服务两个平台：入口不同，大脑相同。这篇文章记录我们实际跑通并稳定运行数周的方案。

## 问题

真正的难点不是"同时在线"，而是三件事：

1. **消息模型不一致**：Telegram 有 reply_to / 话题，Discord 有频道 / thread / 回复，附件和 mention 语法完全不同。
2. **会话归属**：同一个人在两个平台找 bot，算不算同一个对话？上下文要不要互通？
3. **出站差异**：解析模式、长度上限（TG 4096 / Discord 2000）、限流策略各不相同。

## 做法

架构上坚持"适配器薄、核心厚"：

```
TG adapter ─┐                                  ┌─ TG renderer
            ├─ Envelope ─ Agent core (MCP 工具) ─ Router
DC adapter ─┘                                  └─ DC renderer
```

1. **统一信封**：两个适配器都把入站消息压平成同一结构：`platform / chat_id / thread_id / user_id / text / attachments / reply_to`。适配器只做编解码，不碰任何业务逻辑。
2. **会话键**：用 `platform:chat_id:thread_id` 作会话主键，默认跨平台上下文隔离。确实需要打通时，用显式的绑定命令把两个 user_id 关联到同一 profile，而不是靠用户名去猜。
3. **出站渲染**：Agent 产出纯 Markdown，由各平台 renderer 分别转成 Telegram HTML 和 Discord markdown；超长按段落切块，保护代码块完整性。
4. **队列与限流**：每个平台独立出站队列 + 令牌桶，收到 429 按 `retry_after` 退避重试，不做跨平台全局并发。
5. **工具复用**：技能全部走 MCP，Agent 核心不感知平台差异；某技能只想在 TG 开放，用 channel 维度的开关控制。

## 踩坑点

- **MarkdownV2 的转义是重灾区**，直接换 HTML parse mode，故障少一半。
- Discord 的 2000 字符上限：切块要保护代码块；embed 是另一套排版体系，别和正文混用。
- 重连后会收到重复事件，按 `(platform, message_id)` 做幂等去重，否则 bot 会复读。
- TG 超级群的话题和 Discord thread 必须显式写进会话键，漏掉就会串台回复。
- 附件别只存链接：TG 要走 `getFile` 下载，Discord CDN 签名链接会过期，入库前统一落盘。

## 可复用建议

- 会话键设计先于一切，后补最痛。
- 跨平台上下文默认隔离，共享必须显式授权。
- 信封结构加版本号；之后接 Slack、QQ 只是多写一对 adapter/renderer。
- 每条消息带 correlation id 打日志，双平台联调全靠它定位问题。

## 总结

这套路由方案的价值不在"多接了一个平台"，而在于逼你把 Agent 核心和传输层彻底解耦。适配器薄了，核心稳了，第五、第六个平台就只是配置问题。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/43375872b097a750.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/621ffb48a4b770a4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/c641ecfe804d3d8e.png)

