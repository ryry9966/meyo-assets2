---
title: 一个 Agent 同时守两个门：Telegram + Discord 跨平台消息路由实践
feedId: 38854
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

团队沟通天然分裂：一部分人在 Telegram 群里，另一部分在 Discord 服务器。最初的做法是各跑一个 bot，各配一份工具和提示词。结果两个月后就崩不住了——同一件事两边问两遍，TG 里教过的东西 Discord 不知道，工具改一处忘一处。于是做了这次收敛：**一个 OpenClaw 实例，一个 agent 核心，两个通道入口**。

## 问题

拆开看其实是三类：

1. **配置分裂**：两套 bot、两份插件/工具接入，改一次要同步两次；
2. **状态分裂**：会话和记忆各存一份，跨平台上下文完全割裂；
3. **行为分裂**：消息格式、媒体处理、唤醒方式各写一遍，输出风格还不一致。

## 做法

核心思路一句话：**平台差异压缩进最薄的通道层，智能和状态全部收敛到 agent 层**。

1. **单实例双通道**。`openclaw.json` 里同时启用两个通道，Telegram 用 BotFather 的 token，Discord 用 application token：

```json
{
  "channels": {
    "telegram": { "botToken": "***", "allowFrom": ["..."] },
    "discord":  { "token": "***", "allowFrom": ["..."] }
  }
}
```

2. **路由规则收敛到配置**：allowlist 限定允许的 chat；私聊直通，群聊只响应 @提及或显式命令，避免刷屏和误触发。

3. **会话策略：共享记忆，独立会话**。这是整次改造最关键的设计。每个 chat 一个独立 session（用 `通道 + chat id` 作 key），两边对话互不穿插；但 workspace 里的 MEMORY.md、笔记、长期记忆是共享的——TG 里交代过的背景，Discord 那边也能"知道"。

4. **工具挂 agent 层，不挂通道层**。所有 MCP server 统一配置在 agent 上，两个通道天然调用同一套工具，新增能力只需接一次。

5. **出站格式化单独做**：进 agent 前统一归一化（语音转写、图片落盘），出站时按平台适配——Discord 用 embed 加 2000 字符分块，Telegram 走 Markdown，长文分页。

## 踩坑点

- **Discord 不开 Message Content Intent**，bot 收到的消息内容全是空的，连接日志却一切正常，排查了半天。去 Developer Portal 把 intent 打开即可。
- **Telegram 群聊 privacy mode**：默认只收命令消息，群里 @它说话它是聋的。要么在 BotFather 关掉 privacy mode，要么接受"命令驱动"的群聊模式。
- **强行共享会话会翻车**：早期试过让两个通道共用一个 session，结果两边对话在上下文里互相穿插，模型经常答非所问。改成 per-chat session 后立刻正常。
- **回声循环**：agent 的回复触发了对面平台的另一个自动化，形成死循环。加一个 self-id 过滤，自己的消息不进路由。
- **速率限制别赌运气**：Telegram 对 bot 有 flood 控制，长回复直接 429；分块 + 稍作间隔是标准做法。

## 可复用建议

- **通道层做薄**：适配器只做协议归一化，不做业务判断，路由逻辑放配置而不是代码；
- **工具单一事实来源**：MCP 挂 agent，不挂通道，杜绝"两边各接一份"；
- **记住八个字：共享记忆，独立会话**；
- **每条路由决策打日志**（通道、chat id、session key），跨平台排障效率翻倍；
- **每个通道留独立开关**，一个平台出问题可以单独下线，不影响另一个。

## 总结

这次改造的代码增量其实不大，功夫主要花在协议细节和架构取舍上。回头看，跨平台消息路由的难点从来不是"连上两个平台"，而是想清楚**哪些东西该共享、哪些必须隔离**。把平台差异摁在最薄的适配层里，agent 才能真正做到"一个大脑，多个入口"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/277768b98aa0b75d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/dbee32fe92e3747f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/23c9d1dc8b4f7687.png)

