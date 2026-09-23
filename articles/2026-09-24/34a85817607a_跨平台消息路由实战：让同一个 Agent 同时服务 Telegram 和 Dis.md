---
title: 跨平台消息路由实战：让同一个 Agent 同时服务 Telegram 和 Discord
feedId: 38715
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

我们的助手 Agent 最初只挂在 Telegram 上，跑在本地 OpenClaw gateway 上，配了一套工具和 MCP server。后来社群用户开始往 Discord 迁移，问题就来了：是再养一个 Agent，还是让同一个 Agent 同时吃两个平台？显然不想维护两份 prompt、两套工具配置，于是做了一次跨平台消息路由的改造。结论先说：可行，但重点不在"接上"，而在路由。

## 问题

把两个 channel 都打开只是第一步，真正要回答的是三件事：

1. 消息进来后路由到哪个 session？
2. 同一个用户在两个平台都有账号，上下文要不要合并？
3. 回复格式、长度限制、群聊触发条件各不相同，谁来做适配？

## 做法

**Step 1：单 Agent 多 channel。** gateway 配置里 telegram 和 discord 各自填 token，Agent 只定义一份。工具和 MCP server 挂在 Agent 层，两个平台天然共享。示意如下（字段名以你本地版本文档为准）：

```json
{
  "channels": {
    "telegram": { "botToken": "${TG_TOKEN}", "dmPolicy": "allowlist" },
    "discord": { "token": "${DC_TOKEN}", "requireMention": true }
  }
}
```

**Step 2：会话路由策略。** 默认保持"每平台每聊天独立 session"。这是最稳的起点：Telegram 的上下文不会漏进 Discord。如果确实需要跨平台延续，再做定向打通，而不是一上来就合并所有会话。

**Step 3：群聊触发规则。** Telegram 群用 @提及 + 白名单；Discord 用 requireMention + 频道级白名单。原则只有一条：默认不响应，显式触发。

**Step 4：输出归一化。** 系统提示词里不写任何平台专属格式（markdown 表格、@everyone 之类）。格式交给 channel 适配层，长度限制在网关侧切片处理（Discord 单条 2000 字符，Telegram 4096）。

**Step 5：主动消息的默认出口。** 定时任务和 heartbeat 的结果要发回"发起请求的那个 channel"，在配置里显式指定默认投递目标，别让网关猜。

## 踩坑点

- Discord 开发者后台必须开启 MESSAGE CONTENT INTENT，否则群里的消息内容是空的，表现非常像"Agent 装死"，排查起来容易走偏。
- Telegram 不要同时开 polling 和 webhook；群机器人想看到全部消息需要关 privacy mode，但关掉后消息量暴涨，必须配合白名单兜底。
- 共享 session 看着美好，实际等于把 A 平台的聊天记录塞进 B 平台的上下文——既有隐私问题，也浪费 token。我们最后退回了独立 session + 共享记忆文件（挂在 MCP 上），效果反而更好。
- 两个平台同时来消息是常态，确认 session 内消息是排队串行处理的，否则回复会互相穿插。
- 用户身份是平台内的：Telegram 的 user id 和 Discord 的 user id 没有关联，做"私聊发起、结果回推"时要自己维护一份映射表。

## 可复用建议

- 路由决策放在配置里，不要写进 prompt。prompt 是最差的 router。
- 一个 Agent 多 channel 没问题，但一个 session 只服务一个任务上下文。
- 调试时在日志里带上 channel + chat id，路由问题十分钟内能定位；没有这两个字段会查到怀疑人生。
- gateway 配置进 git，每次调整路由规则留痕，出问题能回滚。
- 如果用插件形式接入自定义 channel，沿用同一套 session key 约定，别自创第三套规则。

## 总结

把 channel 当哑管道，把路由当配置，把 session 当上下文的边界。这三个边界划清楚之后，"一个 Agent 服务 N 个平台"就只是多加一份 channel 配置的事情，Agent 本身一行不用改。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/4862fb3b5287e282.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/2b325744f1dfbf15.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/94a69954ce18cb22.png)

