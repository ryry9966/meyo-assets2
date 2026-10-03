---
title: 一个 Agent，两个平台：让 OpenClaw 同时路由 Telegram 和 Discord 的实践记录
feedId: 40302
source: 综合讨论
publishedAt: 2026-10-04
---

## 背景

我此前把 OpenClaw 当私人助理跑在 Telegram 上，后来社区协作挪到了 Discord，于是面临选择：再起一个独立 Agent，还是让同一个 Agent 同时接两个平台。前者的代价是记忆、工具、定时任务各养一份，配置漂移得很快；我想要的是「一个大脑，多个入口」。

## 问题

真正要解决的不是「能不能连上」，而是三件事：

- **会话路由**：同一用户在两个平台是两个 peer，上下文要不要打通；
- **出站格式**：Telegram 的 MarkdownV2 和 Discord Markdown 是两套方言，长度限制也不同；
- **限流与稳定**：两边各有速率限制和重连逻辑，不能让一个平台把另一个拖垮。

## 做法

1. **双通道接入**。在 gateway 配置里同时启用 telegram 和 discord 通道，各自填 bot token。home lab 环境下 Telegram 走 long polling、Discord 走 gateway websocket，都不要求公网 webhook。
2. **单 Agent 绑定**。agents 只定义一个实例，routes 把两个平台的 allowlist（Telegram chat id、Discord guild/channel id）都指向同一个 agent id。
3. **平台差异放适配层，不放 prompt**。出站按目标通道走各自渲染：Telegram 用 HTML parse mode（避开 MarkdownV2 的转义地狱），Discord 按 2000 字符分段，代码块优先保完整。
4. **会话策略做取舍**。我最终选了「每平台独立 session key + 定时摘要」：日常上下文分开，每晚用一次 cron 让 Agent 自己生成跨平台摘要写进记忆文件，第二天两边都能引用。比强行共享 session 干净得多。
5. **媒体统一落盘**。两个通道的图片和文件都下载到 workspace 的 media 目录，Agent 用同一套工具读取，不感知来源。

## 踩坑点

- Discord 开发者后台没开 **MESSAGE CONTENT intent**，bot 能收到消息但 content 恒为空，白排查半小时。
- Telegram MarkdownV2 对下划线、点号都要求转义，改用 HTML parse mode 后问题消失。
- Discord 频道限速约 5 条/5 秒，长回答分段发送必须**串行 + 间隔**，否则排队报 429。
- 断线重连后 Discord 会重放部分事件，按 message id 做了去重。
- heartbeat 和 cron 的输出别广播到两个平台，只在指定 peer 回复，不然会被自己刷屏。

## 可复用建议

- **channel 是传输层，Agent 是大脑**：平台怪癖（转义、分段、intent 开关）全部收敛在通道配置和出站渲染里，system prompt 只描述「你在跟人说话」，不写平台细节。
- allowlist 全部放配置文件并纳入版本管理，别写死在脚本里。
- 日志统一带 `channel + peer` 标签，跨平台排障时能一眼对齐时间线。
- 换 token、换频道先在测试 guild / 测试 chat 跑 dry-run，确认格式渲染无误再切流量。

## 总结

一个 Agent 服务两个平台，工作量的大头不在「路由」，而在出站格式与会话策略。OpenClaw 的通道插件把接入层基本做掉了，剩下的是把平台差异收敛到适配层、把上下文策略想清楚。跑通之后，再加 Slack 之类的通道只是多一份配置，Agent 侧零改动。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/c07ecef35326bb8f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/d0a58fcd7ceab256.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/54d2c63e62ed72ef.png)

