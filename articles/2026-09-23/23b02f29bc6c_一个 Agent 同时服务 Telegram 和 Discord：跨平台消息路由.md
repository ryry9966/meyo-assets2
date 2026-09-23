---
title: 一个 Agent 同时服务 Telegram 和 Discord：跨平台消息路由实践
feedId: 38612
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

社区讨论散在 Telegram 和 Discord 两边，问题重复回答、上下文割裂。最初的方案是给两个平台各跑一个 bot，结果同一套提示词、两份记忆，agent 在两边表现得像两个人。目标很直接：**一个 agent 核心，同一份会话状态和工具集，同时服务两个平台**。

## 问题

表面上看只是“接两个 SDK”，实际是消息模型不一致：

- 会话粒度不同：Discord 有 channel / thread 两级，Telegram 是 chat / topic
- 消息上限不同：Discord 2000 字符，Telegram 4096
- Markdown 方言、mention 语法、媒体下载方式各不相同
- 两个 adapter 各自维护上下文，用户跨平台提问时 agent 会“失忆”

核心矛盾是：agent 核心不该知道平台细节，平台细节又不能污染会话状态。

## 做法

拆成三层：

**1. 适配层**。每个平台一个独立 adapter（可拆成独立进程），只做三件事：收消息、发消息、把原始事件归一化：

```json
{
  "platform": "telegram",
  "session_key": "telegram:chat:123456",
  "msg_id": "9876",
  "user_id": "u_abc",
  "text": "...",
  "media": [],
  "reply_to": null
}
```

**2. 会话路由层**。以 `platform:类型:id` 作为 session key 读写历史，同一 key 进同一 agent 会话，不同 key 完全隔离。router 按 session_key 做简单串行化，避免并发写交错。

**3. Agent 核心**。只认 session_key 和统一消息，工具调用走 MCP，回复统一走 `send_reply(platform, session_key, content)`，对平台无感知。

回复的平台化在适配层完成：长文本按平台上限切段（切在段落边界而非硬截断）、Markdown 降级（Discord 保留代码块标注语言，Telegram 转 `<pre>`）、mention 换成平台格式。

部署上推荐两个 adapter + 一个 router/agent 进程，中间用 Redis Stream（或任意队列）解耦，adapter 崩溃重连不影响会话状态。

## 踩坑点

- **接入方式不对称**：Telegram webhook 需要公网 HTTPS，本地开发用 cloudflare tunnel 顶着；Discord 走 Gateway WebSocket，不需要公网。别按同一套假设写。
- **Discord thread 串台**：thread 里消息的 channel_id 是 thread 的 id 而非父频道。session key 直接用 thread id，另建一层 thread → 父频道的映射用于通知。
- **echo 循环**：不过滤 bot 自身消息会无限自我对话。另外 Discord 默认收不到其他 bot 消息，Telegram 群组 privacy 模式默认收不到普通消息，都要显式配置。
- **两套限流**：Telegram 约 1 msg/s 每 chat，Discord 按 route 分 bucket。回复管线必须带队列和 429 退避，不能裸发。
- **媒体下载不对称**：Discord attachment 是直链；Telegram 要先 getFile，且 Bot API 下载上限 20MB，大文件提前判掉。
- **幂等**：webhook 会重试，用 `platform + msg_id` 做去重键，否则重复触发。

## 可复用建议

- 统一消息模型是地基，宁可字段冗余，也别让平台特有结构漏进 agent
- 先上“只读观察模式”：收消息、记日志但不回复，跑两天确认 session key 粒度和噪音过滤没问题，再放开回复
- 日志统一带 `platform/session_key` 前缀，排障时一条 grep 还原完整链路
- adapter 保持薄、保持可替换，新接 Slack 或其他平台理论上只需新增一个 adapter

## 总结

跨平台路由不是“接两个 SDK”，而是把**消息归一化、会话路由、回复适配**三件事拆干净。adapter 薄、状态集中在 router、agent 核心平台无关，之后换模型、加工具、扩平台都是局部改动。这套结构最大的收益是两边用户共享同一份 agent 记忆，讨论不再分裂。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/426f628e23620d1a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/d523ca39327a7fab.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/ffb098d9c451c0d9.png)

