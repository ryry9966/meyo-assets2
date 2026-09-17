---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 38020
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

我的联系人和社群横跨 Telegram 与 Discord 两侧。最初偷懒跑了两个 Agent 实例分别对接，一周后就扛不住了：两份记忆各说各话、token 开销翻倍、同一个问题得到两个版本的回答，定时任务还会重复触发。这篇帖子记录把架构收敛成"一个大脑 + 两个频道适配器"的过程，全部基于 OpenClaw 的多 channel 能力，没有自研网关。

## 问题

跑通两个平台不难，难的是三件事：

1. **会话隔离**：两个平台的 chat_id 空间互不相干，session key 设计不当会撞键或割裂；
2. **输出适配**：Markdown 方言、长度上限、分段规则各不相同；
3. **身份统一**：同一个人在两个平台是两个 ID，Agent 眼里是"两个陌生人"。

## 做法

**1. 单实例挂双 channel。** Gateway 只跑一份，channels 里同时配置 Telegram（bot token + 轮询）和 Discord（bot token + intents），两个 channel 共享同一个 workspace。记忆、技能、定时任务天然同源，这是整件事里最值钱的一步。

**2. 统一 session key 约定。** 用 `platform:chat_id` 作 key，例如 `telegram:12345`、`discord:98765`。前缀隔离是底线——不要拿裸 ID 拼 key，两平台 ID 空间不同，撞键后极难排查。

**3. 身份映射（可选但建议）。** 在 workspace 维护一张 `discord_uid ↔ telegram_uid` 映射表，并在 system prompt 里注明"这两个身份是同一个人"。做法粗暴但够用；不做映射，至少要让 Agent 明确知道跨平台上下文不共享。

**4. 输出适配层。** Telegram 单条上限 4096 字符，Discord 是 2000，统一按 1800 切块，并避开代码块边界；转义差异（MarkdownV2 vs 标准 Markdown）全部收在各自的 channel adapter 里处理，Agent 层永远只输出标准 Markdown。

**5. 主动消息单一出口。** 心跳、定时任务的主动通知只允许发到主频道（我的场景选了 Telegram），Discord 只做被动响应，避免一个事件在两侧各响一次。

## 踩坑点

- Discord 后台不开 **Message Content Intent**，bot 收到的消息 content 全是空串，现象像"Agent 失忆"，实际是权限问题；
- 误起两个进程同时轮询 Telegram，会互相抢 `getUpdates`，表现为消息随机丢失；
- 切块截断未闭合的代码块时，两侧渲染会同时崩掉，切块前必须检测 ``` 边界；
- Discord 对批量发言限速很凶，广播类逻辑必须加间隔。

## 可复用建议

- **适配器保持薄**：平台差异（转义、切块、限速）全部下沉到 channel 层，核心层对平台保持无知；
- session key 约定第一天就要定对，后期迁移成本极高；
- 先用一个 echo 技能打通双平台收发链路，再上真实技能；
- 日志里每条消息都带 platform 标签，排障效率差一个量级。

## 总结

多平台接入的工程量不在"连接"，而在会话设计与输出适配。"一个大脑、多个嘴巴"的结构经实测是稳的：channel 层处理方言，核心层保持通用。这套配置跑了两个多月，两侧合计日均几百条消息，没有再出现记忆分叉或重复响应。如果你的用户也分散在多个 IM 里，建议从 session key 约定开始设计，而不是从接入开始。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/6d9cef79fc8d9052.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/91830686c8aa7050.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/e0e0a556ef31c7b7.png)

