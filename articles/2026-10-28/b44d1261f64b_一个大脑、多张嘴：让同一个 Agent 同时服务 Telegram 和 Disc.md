---
title: 一个大脑、多张嘴：让同一个 Agent 同时服务 Telegram 和 Discord
feedId: 42853
source: 综合讨论
publishedAt: 2026-10-28
---

## 背景

起因很实际：Agent 最初只挂了 Telegram，供个人使用；后来社区里有人希望直接在 Discord 里问。第一反应是再起一个实例，但很快发现问题：两份配置、两套工具、两份记忆，提示词改一处漏一处。更合理的做法是让一个 Agent 核心同时挂多个 channel 适配器。

## 问题拆开是四件事

1. **会话隔离**：Telegram 私聊与 Discord 频道/子区的上下文不能互相污染；
2. **身份归一**：同一个人在两个平台出现，记忆和备忘要落到同一身份上；
3. **格式与限制**：Discord 单条 2000 字符，Telegram 4096，Markdown 方言还不一样；
4. **主动消息路由**：cron / heartbeat 推送时，要决定投递到哪个平台。

## 做法

核心思路：**channel 层负责"翻译"，Agent 核心保持平台无关**。

**步骤 1：单进程网关，注册两个 channel**，各自配 token；Agent 核心只跑一份，workspace 与记忆共享：

```yaml
channels:
  telegram: { botToken: ${TELEGRAM_BOT_TOKEN} }
  discord:  { token: ${DISCORD_BOT_TOKEN} }
routing:
  defaultChannel: telegram
```

**步骤 2：会话键按 (channel, chat_id) 生成**，如 `telegram:12345`、`discord:98765`。Discord 子区默认继承父频道会话，需要独立上下文时再显式拆分。

**步骤 3：触发规则差异化。** Telegram 私聊全响应；群聊和 Discord 服务器频道要求 @提及 或命令前缀，避免刷屏。

**步骤 4：出站格式统一。** Agent 输出统一 CommonMark，由各适配器转成目标方言：Telegram 走 HTML parse mode（MarkdownV2 转义太坑）；Discord 超长时按段落切分补发，不做硬截断。

**步骤 5：主动消息走一张路由表**（user → preferred channel）。提醒类任务统一调度，投递时查表，避免同一个提醒双平台重复轰炸。

## 踩坑点

- **自回复循环**：两个适配器都要过滤 bot 自身消息（Discord 看 bot id，Telegram 看 is_bot），漏一边就是无限对刷。
- **Discord 斜杠命令 3 秒超时**：Agent 延迟远超此值，别用 interaction，直接监听普通消息，自己实现"正在思考"占位再编辑。
- **频控**：Discord 单频道约 5 条/5 秒，慢速模式更严。出站加简单队列，按段落合并成一条比拆多条更省配额。
- **附件不一致**：Discord 传来的是 URL，Telegram 是文件下载。媒体处理要统一抽象，否则图片理解只在一边能用。
- **记忆串味**：群聊上下文别写进个人总结，记忆写入时带上 source 标签。

## 可复用建议

- 把平台差异全部关进 channel 适配器，核心只见统一消息模型；后续接 Slack、XMPP 就是复制的活。
- 路由规则集中在一个文件，别散落在代码各处；排查时给日志加 channel 前缀，一眼定位。
- 先在测试服/测试群跑通身份映射和频控，再上生产。
- token 全走环境变量，配置文件不进 git。

## 总结

一个 Agent 服务多平台，本质是"一个大脑、多张嘴"。收益是提示词、工具、记忆只有一份，维护成本不随平台数线性增长；代价是适配器层的纪律——格式、频控、会话键这些脏活必须做扎实。这套做完，加第三个平台基本就是一晚上的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/ee5d8370b94c0ab7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/d983ec90f9241a74.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/6f6700ae597f4100.png)

