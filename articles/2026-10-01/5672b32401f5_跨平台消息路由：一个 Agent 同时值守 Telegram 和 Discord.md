---
title: 跨平台消息路由：一个 Agent 同时值守 Telegram 和 Discord
feedId: 40017
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

我们最初只在 Telegram 上跑了一个 OpenClaw Agent，负责值班提醒和日常问答。后来团队一部分人迁到 Discord，问题就来了：是再部署一个实例，还是让一个实例同时接两个平台？折腾了两周，把路由层理顺之后，总结成这篇。

## 问题

最常见的两种做法都有坑：

- **跑两个 Agent 实例**：两份记忆、两份工作区，同一个问题在两边得到不同答案，配置和插件维护成本直接翻倍。
- **两平台共用一个会话**：在 Telegram 里聊到一半，回复突然引用了 Discord 上另一个频道的上下文，信息串台，用户体感很差。

本质问题是传输层（channel）和智能体核心（agent core）没有分层。多平台的价值是"一个大脑、多个入口"，而不是"多个大脑"。

## 做法

1. **单实例多通道**。一个 OpenClaw 实例，同时启用 Telegram（bot token 长轮询）和 Discord（gateway bot）两个 channel 插件。核心配置只写一份，不要复制目录再改。
2. **设计好会话键**。路由正确与否的关键在 session key。我们采用 `platform:chat_id` 作为默认粒度——每个群的会话互相隔离，但共享同一份工作区文件和长期记忆；私聊则用 `platform:user_id`。
3. **统一内部消息模型**。入站消息先归一化成五个字段：sender、platform、chat、text、attachments。channel 插件只做协议转换，不夹带任何业务逻辑。
4. **出站渲染分离**。回复时按目标平台走不同 renderer：Telegram 走 HTML 解析模式，Discord 走自己的 markdown 和 embed。Agent 核心只输出结构化内容，不直接输出平台方言。
5. **身份映射（可选）**。如果同一个人两边都有账号，维护一张映射表让跨平台会话可以接续。没这个刚需就别做，复杂度不值。

## 踩坑点

- Telegram 的 MarkdownV2 转义是地狱，字符集之外一概要转义。我们最终放弃它，全走 HTML 模式，渲染失败率归零。
- Discord bot 在线但收不到任何消息内容，八成是没开 Message Content Intent。
- 同一个 Telegram token 被两个进程同时长轮询会报 409 冲突，重启前确认旧进程真的死了。
- 忘了过滤 bot 自身和其他 bot 的消息，两个 bot 互相触发，一晚上烧掉十几万 token。入站过滤器必须默认忽略一切 bot 消息（白名单除外）。
- Discord 单频道限速很紧，突发流量要进出站队列；Telegram 是每秒每聊天 1 条。没队列的话高峰期日志里全是重试。
- 文件大小限制两边不同（Telegram Bot API 下载 20MB，Discord 也有上限），大文件直接落工作区发链接，别走消息中转。

## 可复用建议

- channel 层保持"薄"：只做传输、格式化、鉴权，业务逻辑一个字都不要放进去。
- 路由规则和身份映射放进配置文件并纳入版本管理，不要硬编码。
- 日志里每条消息打上 platform 标签，出问题时能一眼定位是哪个通道。
- 先在一个测试群里灰度跑一周——会话键的设计几乎一定要返工一次，返工发生在测试群比发生在生产群里好。

## 总结

多平台接入的工程量，90% 花在路由正确性上：会话隔离、身份识别、格式适配。OpenClaw 的 channel 架构本身支持这种分层，关键是管住自己，别把逻辑塞进 channel 插件。"一个大脑、多个入口、薄适配层"——这套结构后来我们加第三个平台时，基本零改动。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/a636604cb7cc3512.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/26b291515254b025.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/028dd055cdf50f43.png)

