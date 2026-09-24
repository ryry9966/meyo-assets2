---
title: 跨平台消息路由：让同一个 Agent 同时服务 Telegram 和 Discord
feedId: 38741
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

我们的 Agent 最初只挂在 Telegram 上，服务内部群：查数据、跑脚本、定时汇总。后来另一拨同事常驻 Discord，能不能让同一个 Agent 两边都接？结论是可以，OpenClaw 天然支持多通道并存——难的不是"连上"，而是把会话、格式和限速这些差异抹平。

## 问题拆开看

核心冲突有四个：

1. **会话隔离**：Telegram 靠 chat id，Discord 靠 channel id，是两套会话键。上下文天然不互通——这是特性不是 bug，群里聊的不该串到频道。
2. **消息格式**：Telegram 的 MarkdownV2 转义极其挑剔，Discord 是宽松 markdown 还支持 embed。同一段回复裸发，一边渲染崩、一边没事。
3. **长度与限速**：Discord 普通消息 2000 字符，Telegram 4096；Discord 还有频道级限速，批量输出容易撞 429。
4. **触发语义**：Discord 靠 @ 提及，Telegram 群里常用触发词或 / 命令，两边行为要对齐。

## 做法

**第一步：单实例、双通道。** 一个 gateway 进程里同时启用 `channels.telegram` 和 `channels.discord`，各自填 token。关键点是两个通道绑到同一个 agent——skills、MCP server、workspace 全部共享，不用跑两份。

**第二步：各自的 allowlist。** Telegram 按 chat id 白名单，Discord 按 guild/channel 白名单。只放行该放的地方，避免 bot 被拉进公开服务器后变成公共接口。

**第三步：输出走通道适配。** 长回复按平台分片（Discord 2000、Telegram 4000 留余量）；Telegram 侧统一用 HTML 或纯文本模式，绕开 MarkdownV2 转义坑；代码块在 Discord 正常渲染，Telegram 里贴文件链接更稳。

**第四步：统一身份与日志。** 建一张发送者映射表，把 Discord 用户和 Telegram 用户对应到同一套内部 ACL；日志加 `[tg]`/`[dc]` 前缀，排障一眼分清来源。

**第五步：探活。** 用 cron/心跳任务让 Agent 定时自检，两个通道各发一条探活消息，任何一侧掉线能第一时间知道。

## 踩坑记录

- **Discord 收到的消息内容是空的**：开发者后台没开 MESSAGE CONTENT INTENT。gateway 连上了，但 content 全空。最常见也最隐蔽。
- **Telegram 409 冲突**：本地调试拉过一次 getUpdates，服务器实例就开始报 conflict。同一 token 只允许一处拉取，调试完杀干净本地进程。
- **Discord thread 的 channel id 是新的**：白名单按频道 id 写死后，thread 里 bot 直接失聪。按 category 或 guild 维度放行更省心。
- **长任务并发互踩**：两边几乎同时触发同一个跑批技能，临时文件互相覆盖。给临时目录加会话前缀，并串行化重工具调用后解决。
- **语音消息**：Telegram 语音是 ogg/opus，Discord 附件是直链 URL，转写管道要按来源分叉处理。

## 可复用的建议

- 通道层保持"薄"，只做收发和格式适配；业务逻辑全部收在 skills/MCP 里，加平台只改配置不改逻辑。
- token 走环境变量，配置文件才能安心进仓库。
- 灰度顺序建议：先让 Discord 单向"同播"（只发不收），验证格式和限速，再放开双向。
- 每个通道写一条端到端冒烟用例：发一条含代码块、长文本、图片的消息，验证分片和渲染。

## 总结

一个 Agent 服务 N 个平台，架构上不难，工作量都在"差异抹平"：会话键、转义规则、长度、限速、触发方式。把通道适配做薄、把身份和日志做统一，之后再加 Slack、飞书，基本就是改配置的活了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/64c7e048cc11bbaa.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/6fa457f9f9d957ca.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/dba7a30c73c40c7f.png)

