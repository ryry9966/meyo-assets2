---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 39676
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

我的机器人最早只挂在 Telegram，群里用得顺手。后来团队讨论主阵地搬去 Discord，第一反应是再起一个 bot——很快就被两份配置、两套会话记忆、两边 prompt 改不同步搞烦了。目标于是很明确：**同一个 Agent、同一份记忆和工具集，两个平台只是不同的入口**。

## 问题

表面上是"多接一个 channel"，实际差异在四层：

- **消息格式**：Telegram 4096 字符上限 + MarkdownV2 转义地狱；Discord 2000 上限，有自己的 markdown 和 embed。
- **身份模型**：同一人在两个平台的 user id 完全不同，session 算一个还是两个？
- **触发方式**：Telegram 群靠 @ 或命令，Discord 靠 mention 和 slash command，触发词不通用。
- **主动消息**：定时任务、心跳产生的内容，该发给谁？

## 做法

核心思路：**channel 层只做收发与协议适配，业务逻辑全部收敛在 Agent 侧**，路由策略集中在 gateway 配置里。工具和 MCP server 都挂在 agent 上，两个平台天然共享，不用重复配。

1. **双 channel 接入**。两个 channel 绑到同一个 agent，workspace 与记忆共享（不同版本字段名略有差异，以官方文档为准）：

```json
{
  "channels": {
    "telegram": { "botToken": "env:TELEGRAM_BOT_TOKEN" },
    "discord":  { "token": "env:DISCORD_BOT_TOKEN" }
  },
  "agents": {
    "default": { "workspace": "~/agent-workspace" }
  }
}
```

2. **会话键设计**：session key 用 `channel:chatId`，如 `telegram:12345` 与 `discord:98765` 是两个独立会话。要打通就在 agent 侧维护 identity map（手动登记或绑定命令），把两个 id 指向同一 user。建议默认不打通，按需打通，避免两边上下文互串。

3. **出站适配**：所有回复先过统一 renderer，按目标平台降级——长文在 Discord 走分片或附件，在 Telegram 按 4000 字符切块；MarkdownV2 转义只写一个函数，别散落各处。

4. **主动消息显式路由**：定时任务必须指定 `route`（如 `telegram:me`），不给默认值——宁可不发，也不要随机发进某个群。

5. **灰度上线**：Discord 先只开私聊 + 一个测试频道，观察一周再放开群聊。

## 踩坑点

- **MarkdownV2 漏转义**：`_ * [ ]` 没处理会导致 Telegram 直接 400，且整条消息失败。统一 escape + 失败降级为纯文本重发。
- **Discord rate limit**：分片发送要加间隔（约 1 条/秒），批量通知时尤其明显，收到 429 必须退避。
- **会话强串**：起初把两平台 session 强行 merge，结果 Agent 会引用"另一个群里"的上下文，观感很怪。改成**共享长期记忆文件、保留独立会话**后体验立刻正常——这可能是本篇最重要的一条。
- **心跳消息忘配 route**：定时汇报发进了测试群，被群友围观了一晚上。
- **线程语义不对齐**：Discord thread 和 Telegram 话题别指望一一映射，按"回复即跟帖"处理最省事。

## 可复用建议

- channel 适配层写薄：只做协议转换，格式化逻辑集中到一个 renderer。
- 出站日志带 channel 标签，排障时一眼区分来源。
- 发送加幂等 key，webhook 重试不会重复推送。
- 两平台开关做成独立 feature flag，单边故障不影响另一边。

## 总结

跨平台的关键不是"多接一个 bot"，而是把**身份、会话、格式、路由**四个问题想清楚再动手。gateway 式架构天然适合这件事：channel 可插拔，Agent 保持单一事实源。落地顺序建议是：先单平台私聊跑通 → 再放群聊 → 最后打通身份映射。顺序反了，每一步都会多踩几个坑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/cff91d7d416d705f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/e070d32ac3de8725.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/f05dc8a2d01558e1.png)

