---
title: 一个 Agent 服务两端：Telegram × Discord 消息路由实践
feedId: 41965
source: 综合讨论
publishedAt: 2026-10-19
---

## 背景

我们的场景比较典型：核心用户在 Telegram 群，社区讨论在 Discord 服务器。最早是两个独立 bot、两套脚本，结果同一个用户在两边问同一个问题，Agent 给出两个不同答案，上下文互不相通。于是把两平台收敛到一个 Agent 实例后面，做统一的消息路由。

## 问题

拆开看其实是四个子问题：

1. **接入差异**：Telegram 用 long polling 就能跑，Discord 走 WebSocket gateway，且必须单独开启 MESSAGE CONTENT INTENT 才能读到正文。
2. **消息格式差异**：Markdown 方言不同、长度上限不同（Telegram 4096 / Discord 2000）、附件与回复引用结构也不一样。
3. **会话归属**：哪些聊天共享同一个 Agent 上下文，哪些必须严格隔离。
4. **身份映射**：同一个人在两边是两个不同的 ID。

## 做法

架构上采用 adapter + 统一中间表示：

- 每个平台一个轻量 adapter，只负责收消息、归一化（sender / text / attachments / reply_to）、发消息；
- Agent 核心只处理标准化消息对象，完全不感知平台；
- 路由用一张配置表声明 `chat_id -> session_key`，Telegram 群 A 和 Discord 频道 B 既可以指向同一个 session（共享上下文），也可以各自隔离；
- 输出侧做两个小模块：按块边界的长度分片，和 Markdown 方言转换。

落地步骤：

1. 先跑通单平台（Telegram polling），验证 session 逻辑；
2. 加 Discord adapter，开齐 intents；
3. 抽公共的 normalize / send 接口，把平台细节全部下沉到 adapter；
4. 上路由表，分别验证「共享上下文」和「完全隔离」两种 case；
5. 最后加幂等去重（平台 message id 做 key）和每通道独立的发送队列。

## 踩坑点

- **Telegram MarkdownV2 转义极易出错**，最后退回 HTML parse mode，代码块场景稳定很多；
- **Discord 忘开 MESSAGE CONTENT INTENT**，bot 一直收到空文本，排查了半小时才发现是权限问题；
- 按 2000 字符硬切会把代码块切烂，必须按段落/代码块边界切，再补围栏；
- Discord 单频道 5 条/5 秒限速，群发通知时要做合并与排队，否则 429 刷屏；
- session key 一开始用了全局 user id，两个群的上下文串台，改成 channel + thread 粒度才干净；
- 身份映射表要支持人工合并，纯自动 heuristic 合并出过错。

## 可复用建议

- **adapter 越薄越好**，平台差异绝不能漏进 Agent 核心；
- 路由规则配置化，别写死在代码里，改路由不该需要重新部署；
- 每个 channel 单独记录延迟和失败率，路由问题要能按平台归因；
- 默认隔离、显式共享，反过来很容易把 A 群的内容带进 B 群。

## 总结

跨平台路由本身不难，难的是把「接入差异」和「会话语义」分成两层：adapter 吸收协议与格式差异，路由表决定上下文归属，Agent 核心保持平台无关。这套结构后来接入新平台时，基本只需要新写一个 adapter，核心零改动。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/e292ee8e3f2bb785.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/e3518731dfea2341.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/ceb72ec7e61d1e03.png)

