---
title: 一套大脑，两个门口：让同一个 Agent 同时接住 Telegram 和 Discord
feedId: 41228
source: 综合讨论
publishedAt: 2026-10-11
---

## 背景

我们的 Agent 原本只跑在 Telegram 群里，后来一部分用户迁到了 Discord。第一反应是复制一份配置再开一个 bot，结果两周内就养出了"两个人格"：记忆不通、工具配置各自漂移，同一个问题在两个平台能拿到不同答案。这才停下来重新做路由设计。

## 问题的本质

多平台接入不是"多配一个 token"的事，核心矛盾有三个：

1. **会话与记忆归属**：同一份上下文，还是各平台各一份？
2. **配置重复**：MCP 工具、system prompt、allowlist 维护两套，迟早漂移。
3. **平台差异**：Discord 单条 2000 字符、Telegram 4096；Markdown 方言不同；触发方式不同（@mention vs 命令）。

## 做法

**1. 单 Agent 挂双通道。** OpenClaw 的模型是 agent core 对接多个 channel adapter。Telegram 和 Discord 各配 token，但 workspace、工具、prompt 只有一份：

```jsonc
// ~/.openclaw/openclaw.json（简化示意）
{
  "agents": { "main": { "workspace": "~/agents/main" } },
  "channels": {
    "telegram": { "botToken": "${TG_TOKEN}", "allowFrom": ["..."] },
    "discord":  { "token": "${DC_TOKEN}", "allowFrom": ["..."] }
  }
}
```

**2. 会话键按 `channel:userId` 复合隔离。** 对话上下文不跨平台串，但长期记忆（memory 目录）共享一份，保证事实类知识一致。

**3. 触发规则分通道。** 群聊里 Telegram 用 `/ask` 或回复触发，Discord 用 @mention；私聊一律直答。allowlist 按通道分别配置，避免一边泄漏另一边跟着开放。

**4. 出口做格式归一。** 代码块、加粗在 outbound 层转成各平台方言；超长消息先切分再投递；工具调用预计超过 3 秒，先回一个"处理中"占位，防止平台侧超时或用户重复触发。

## 踩坑点

- **Discord 读不到消息正文**：@mention 能收到但 Agent 没反应，排查半天是没开 Message Content Intent。
- **Telegram 409 冲突**：本地调试用 polling，服务器上残留旧 webhook，`getUpdates` 一直报错。换环境记得先 deleteWebhook。
- **会话键不能用用户名**：Discord 改昵称后"会话换人了"，必须用稳定 ID。
- **MarkdownV2 的转义地狱**：下划线、句号都要转义，干脆统一走 HTML parse mode，坑少很多。
- **广播式回复撞限速**：同时投两个平台触发 Discord rate limit。改成只回复消息来源通道，跨平台通知走显式指令。

## 可复用建议

- **通道层保持薄**：只做鉴权、格式转换、限速；所有判断逻辑上收到 agent，否则双通道会演化成双套业务。
- **身份映射单独维护**：建一个 `identity.json` 存 tg id ↔ dc id 的对应关系，为将来合并记忆留口子。
- **排障分层走**：先看通道日志确认消息是否到达，再看会话文件确认是否处理，最后才怀疑模型。
- **每通道配健康自检**：定时发一条自检消息，token 失效第一时间知道，不至于半个平台静默挂掉一天。

## 总结

一套大脑、两个门口完全可行，前提是把平台差异锁死在适配层，把身份与会话模型想清楚再动手。上线一个月后，两边答案不一致的问题基本消失，维护成本并没有翻倍，主要开销集中在最初的格式归一层。如果你也在多个平台养 Agent，建议先画清楚"什么共享、什么隔离"这张表，再写第一行配置。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/d41d7932755ae70b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/7f5ab19182a7c74d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/4808916e71f5a70b.png)

