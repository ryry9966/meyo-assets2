---
title: 一个 Agent 同时服务 Telegram 和 Discord：消息路由实践
feedId: 42772
source: 综合讨论
publishedAt: 2026-10-27
---

## 背景

我的 OpenClaw 最初只接了 Telegram，自己用一直很稳。后来团队日常协作迁到 Discord，总有人问"机器人能不能也拉过来"。第一反应是再起一个实例单独接 Discord，试了两天就放弃：两份配置、两套会话记忆，同一个问题在两边答案不一致，维护成本直接翻倍。于是回到单实例多通道的路线上，把"一个 Agent 同时服务两个平台"做成了常态，稳定跑了一个多月，这里记录一下做法和踩过的坑。

## 问题

拆开看是三件事：

1. **消息进出**：Telegram 走长轮询，Discord 走 Gateway WebSocket；消息结构、Markdown 方言、长度限制全都不同。
2. **会话归属**：一条消息进来落到哪个会话？回复送回哪里？同一个人在两个平台算不算同一上下文？
3. **主动消息**：定时任务、心跳类推送发到哪个平台，才能不重复轰炸。

## 做法

**1. 单核心 + 薄适配器。** Agent 核心（模型调用、工具、记忆）只有一份，Telegram 和 Discord 作为两个 channel 插件同时启用。适配器只做三件事：收消息、归一化、发消息。配置示意（简化，字段以本地版本为准）：

```json
{
  "channels": {
    "telegram": { "botToken": "...", "allowFrom": ["me"] },
    "discord":  { "botToken": "...", "respondTo": ["mention", "channel:<id>"] }
  }
}
```

**2. 统一信封 + 会话键。** 入站消息归一化成统一信封：channel、sender、sessionId、text、attachments、replyTo。会话键定为 `channel:会话id`——Telegram 用 `telegram:<chatId>`，Discord 用 `discord:<channelId 或 threadId>`。刻意不做跨平台身份合并，两边各聊各的，先保证上下文不串。

**3. 出站走单一路径。** 所有回复携带路由元数据（目标 channel + 会话 id），交给统一的发送模块：按平台拆长消息（Telegram 4096 / Discord 2000）、转义格式、处理 429 退避。异步长任务执行完再回复时，路由信息随任务快照一起走，避免回复落错地方。

**4. 差异化规则。** Discord 仅在被 @ 或指定频道时响应；Telegram 私聊全响应、群组白名单。定时推送固定走 Telegram 这条主通道，Discord 保持安静。

## 踩坑点

- **会话键粒度**：最初只按用户 ID 建会话，两个平台对话互相污染——Discord 里聊部署，回到 Telegram 它接着部署话题说。改成 `channel:chatId` 才干净。
- **格式转义**：Telegram 的 MarkdownV2 转义是无底洞，代码块、下划线频繁炸消息。发送模块统一切到 HTML parse mode 后才稳定。
- **路由元数据丢失**：异步工具执行完，回复被默认发到"最后活跃"的通道，出现过 Discord 提问、答案发去 Telegram 的乌龙。
- **多进程冲突**：两个进程对同一 Telegram token 长轮询会 409；Discord 同 token 双 Gateway 连接也会被踢。单实例单进程是前提。
- **限流**：Discord 频道级 + Telegram 全局限流，出站必须排队重试，否则高峰期静默丢消息。

## 可复用建议

- 入站归一化、出站统一发送，适配器保持薄，业务逻辑永远不要感知平台差异。
- 会话键显式化并写进日志，路由出问题一查便知。
- 主动推送只选一个主通道，克制比全覆盖更重要。
- 上线前用 dry-run（只记日志不发送）跑几天，比任何测试都管用。
- 跨平台身份合并收益小、上下文串台风险大，真有需要走显式绑定指令，不要自动猜。

## 总结

多通道接入本质不是连接问题，而是路由和边界问题：会话怎么切、回复怎么回、主动消息去哪。这三件事用明确的机制定下来，剩下的就是格式和限流的体力活。这套结构目前每天稳定处理几百条双平台消息，没再出现串台和错投。如果你也在做类似的多通道路由，欢迎聊聊各自的会话键设计。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/5d4bdd95c1c2fc79.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/d150a3e647ec82c3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/08103d1690479b17.png)

