---
title: 一个 Agent 吃下两端流量：Telegram + Discord 跨平台消息路由实践
feedId: 39545
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

我们的用户一半在 Telegram，一半在 Discord。最早的方案是偷懒：跑两个独立 Agent 实例，各接各的渠道。两周后就维护不动了——提示词改一遍要同步两处，记忆完全割裂（同一个用户在 TG 说过的偏好，切到 Discord 全忘），排障要盯两份日志。

## 问题

拆开看其实是三层：

1. **传输层**：两个平台的协议、鉴权、限流机制完全不同（Telegram 长轮询/webhook，Discord 走 gateway websocket）；
2. **会话层**：哪些上下文共享、哪些隔离；
3. **表达层**：Markdown 方言、媒体附件、消息长度限制都不一样。

## 做法

核心思路一句话：**渠道只做哑管道，Agent 只见归一化后的信封。**

**Step 1：单网关双渠道。** 一个 gateway 进程，在 `openclaw.json` 里同时启用两个 channel（示意，字段名以你的版本为准）：

```json
{
  "channels": {
    "telegram": { "botToken": "<TG_TOKEN>", "dmPolicy": "allowlist" },
    "discord":  { "token": "<DC_TOKEN>", "guilds": ["<guild_id>"] }
  }
}
```

改完 `openclaw gateway restart`，两端一起生效。DM 和服务器准入都用白名单收紧，别裸奔。

**Step 2：归一化信封。** 消息进 Agent 前，统一成 `{platform, user_id, text, media[], reply_to}` 结构，并在系统提示词里追加一段平台元信息，让 Agent 知道自己在和哪个平台的用户说话。仅这一步就解决了 80% 的"语气和格式水土不服"。

**Step 3：媒体统一落盘。** Discord 的附件 URL 是带签名、会过期的，Telegram 的文件要走 bot API 拉取。加了一个前置步骤：任何附件先下载到 workspace 的 media 目录，Agent 拿到的永远是本地路径。这是整个方案里最值钱的抽象。

**Step 4：会话策略。** 默认 session 按"渠道 + 会话对象"隔离，两个平台天然不串。我们保留了隔离，需要时通过显式指令让 Agent 读另一边的摘要。强行共享一个 session 试过一次，两边对话互相渗透、身份混乱，第二天就回滚了。

## 踩坑点

- **Telegram 的 MarkdownV2 转义是黑洞**，`_ * [ ] \` 都要处理。最后直接切 HTML parse mode，世界清净了。
- **限流两边都要排队**：Discord 按路由桶算，长回复分片容易被 429；Telegram 是 per-chat 1 msg/s。统一套了发送队列 + 指数退避。
- **同进程共生的风险**：一个渠道插件 panic 会把另一个一起带重启。日志里务必打上 channel 标签，否则根本分不清是谁挂的。
- **Typing 指示不通用**：Discord 的 typing indicator 和 Telegram 的 chat action 触发时机不同，别指望一套逻辑通吃。

## 可复用建议

- 把所有平台差异关死在适配层，Agent 侧永远只处理归一化信封。以后要加 Slack 或飞书，只是多写一个 adapter。
- 每个平台留一个 `/ping` 健康检查命令，配两个小号做冒烟测试，上线前跑一遍。
- 跨平台同一个人的身份打通，靠维护一张 user map，别让 Agent 猜。
- 广播类消息不要无脑双发，先按各平台的限流预算排队出。

## 总结

这套方案的收益不是"同时上两个平台"这个数字，而是传输和智能彻底解耦之后：提示词只有一份、记忆只有一份、排障只有一份日志。在 OpenClaw 里，渠道就应该被当成可热插拔的哑管道——哪个平台来了流量就插哪个，Agent 本身一行不动。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/db98e1071a604878.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/e4e0ea57f4e79fe5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/2d8ab3d429e693cb.png)

