---
title: 一个 Agent 值两班岗：Telegram + Discord 双通道路由实录
feedId: 39321
source: 综合讨论
publishedAt: 2026-09-28
---

## 背景

最早只是在 Telegram 群里挂了一个 OpenClaw Agent 做值班问答，跑稳之后，Discord 那边的同事不想再切软件来提问。第一反应是再起一个实例，但很快发现两份 MEMORY、两套工具配置，同一个问题两边答着答着就分叉了。于是改成单实例双通道，本文记录这套路由的实际做法。

## 问题：不是跑两个实例那么简单

- **触发行为不同**：Telegram 群默认要被 @ 或被回复才响应；Discord 靠 mention / 前缀，规则各一套。
- **格式方言不一致**：Telegram 的 MarkdownV2 转义极其苛刻，Discord 有 2000 字符硬上限（Telegram 是 4096）。
- **Session 归属**：两个平台的 chatId 都是数字，裸拼 session key 会撞。
- **身份问题**：同一个人在两个平台，算不算同一个上下文？

## 做法

**1. 单实例开两个通道插件。** OpenClaw 的通道层本质是适配器，把两个平台接进同一个 gateway，Agent 工作区、MEMORY、MCP 工具全部共享。不要部署两个进程，否则记忆同步是持续成本。

**2. 统一入站信封。** 所有消息进模型前先归一成 `{channel, chatId, userId, text, attachments}`，路由只认这个结构。session key 强制带 channel 前缀（`telegram:12345` / `discord:67890`），避免 ID 空间冲突。默认按“每群一个 session”隔离。

**3. 身份合并缓一步。** 我们最初想直接共享 session，结果两边话题互相污染。改成“共享长期记忆、隔离会话上下文”，再在 Agent 侧挂一张 userId 映射表，把同一自然人两个平台的 ID 对齐到同一 profile，粒度细、可控。

**4. 出站方言层。** Agent 只输出标准 Markdown，由各通道适配器转译：Telegram 用 HTML parse mode（躲开 MarkdownV2 的转义地狱），Discord 保留 Markdown；长消息按平台上限切片，且感知代码块边界。

**5. 触发规则分开配。** Telegram 群保持“被提及或被回复才响应”；Discord 用 mention 触发、私信全响应。两边都把 bot 自身消息排除出处理队列，防回环。

## 踩坑点

- **切片最坑**：按字符数硬切会切断代码块围栏，Discord 渲染直接崩。先按块（段落/代码块）聚合，块本身超限再降级切。
- **Telegram 群触发放宽成“全部响应”后会刷屏**，很快被管理员请出去；严格 mention 才能长期共存。
- **Discord 附件 URL 有时效**，消息里的图片链接要尽快转存，否则隔天 404。
- **限速**：Telegram 单群约 1 msg/s，连续回复要排队，适配器里加个简单队列比事后补救省事。

## 可复用建议

- 通道层保持“笨”：只做归一化和转译，智能全放 Agent 一侧。以后接 Slack、飞书只是加适配器。
- 日志永远带 channel 标签，跨平台排障时这是唯一能快速定位的线索。
- 切片器写成独立模块并配单测：输入超长 Markdown，断言输出块不破坏代码围栏。
- 先隔离、后合并：session 隔离是安全默认值，身份合并由业务需求驱动再做。

## 总结

一个 Agent 服务两个平台，代码量其实不大，关键在边界设计——**入站归一、出站方言、session key 纪律**。这三件事定清楚之后，接入第三个通道基本是增量工作，而不是重构。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/9fe25efca08f4582.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/b53f3c1a909de708.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/cc88379ee5d013d1.png)

