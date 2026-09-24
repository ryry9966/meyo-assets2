---
title: 跨平台消息路由：让一个 Agent 同时值守 Telegram 和 Discord
feedId: 38824
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

最初只在 Telegram 上跑 OpenClaw，团队协作迁到 Discord 后又不想维护两套 Agent——两份配置、两份记忆、两边结论还不一致。目标很朴素：同一个 Gateway、同一个 Agent workspace，两个前端各自收发。

## 问题

真正难的不是"能不能连上"，而是三件事：

1. **会话归属**：不同平台、不同群的消息必须路由到正确的 session，不能串台。
2. **输出适配**：两个平台的 Markdown 方言、长度上限（Telegram 4096，Discord 2000）、媒体行为都不一样。
3. **触发规则**：两边群的"@ 才回"逻辑要分别配置，否则要么刷屏要么装死。

## 做法

**1. 双通道接入。** Gateway 配置里同时声明 `channels.telegram` 和 `channels.discord`，分别填 botToken 和 token。两个 channel 由同一个 Gateway 进程托管，Agent 只有一份。切忌起两个实例——Telegram 的 getUpdates 轮询会互相抢，Discord 网关也会重复回复。

**2. 路由与会话隔离。** 用 allowFrom / 群组白名单控制谁能触发，DM 侧走 pairing 首次配对。session key 默认按 channel + 会话 ID 区分，Telegram 用户和 Discord 用户天然是两个身份——不要试图跨平台合并身份，除非你显式做了映射表。

**3. 触发与格式。** Discord 侧开启 requireMention；Telegram 群依赖 privacy mode（记得在 BotFather 里关掉，否则群里只能收到 /command）。Agent 输出统一写中性 Markdown，出站适配层负责转换；在系统提示里写明"表格只在 Telegram 用、单条回复控制在 1800 字符内"这类硬约束，比事后截断体面得多。

**4. 验证矩阵。** 至少跑一遍：两平台 DM、两平台群内 @、跨平台同时提问、图片收发。确认 session 互不污染，回复各自落在正确位置。

## 踩坑点

- Telegram bot 在群里收不到普通消息，九成是 privacy mode 没关。
- Discord 2000 字符上限，长回答直接截断；让 Agent 学会分段或压缩输出。
- 代码块、表格在两边渲染差异很大，测试别只看一边。
- 双开 Gateway 会重复回复，日志特征是同一消息被处理两次。
- 心跳/定时任务会广播到所有绑定 channel，注意别深夜打扰两边群。

## 可复用建议

- 把 channel 当**薄传输层**，路由逻辑收敛在 Gateway 配置一处，Agent workspace 保持平台无关。
- Token 全部走环境变量，配置文件进版本管理。
- 每个 channel 保留原始 message ID 日志，排障时能对上下游。
- 平台差异写进**格式约束**，而不是为每个平台维护一套系统提示词。

## 总结

一个 Agent 服务多平台在 OpenClaw 里是原生能力，工作量集中在会话隔离和格式适配两个细节上。把跨平台身份当独立实体处理、把格式约束前置到提示层、用验证矩阵收尾——双平台值守就能稳定跑起来，后续接 Slack 之类的新 channel 也只是加一段配置的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/640c0c58ad3c0ddf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/9ef8af4d1186f5eb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/1868cf23ff5c48b4.png)

