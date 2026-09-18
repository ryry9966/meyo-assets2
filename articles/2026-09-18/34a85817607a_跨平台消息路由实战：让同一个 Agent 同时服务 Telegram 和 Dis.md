---
title: 跨平台消息路由实战：让同一个 Agent 同时服务 Telegram 和 Discord
feedId: 38073
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

社区里不少人的 Agent 最初只挂在一个平台上：要么 Telegram bot，要么 Discord bot。用户分散之后问题就来了——同一套 prompt、同一批 MCP 工具，却要维护两个入口，会话状态各自为政，行为还不一致。我最近把两个平台接到了同一个 Agent 核心上，核心思路就一句：**平台差异全部关在适配层里，Agent 核心只认一种内部消息格式。**

## 问题

直接双开两个 bot 看似省事，实际有几个硬伤：

- 两套会话状态，用户在 A 平台聊过的上下文，到 B 平台全丢
- prompt 和工具配置改两遍，迟早漂移
- 消息格式、长度限制、限流策略完全不同，业务代码里到处是 `if platform == ...`

根本原因是把"平台协议"和"Agent 逻辑"耦合在了一起。

## 做法

### 1. 先定义规范消息（canonical message）

定最小字段：`channel`、`user_id`、`chat_id`、`thread_id`、`text`、`attachments`、`reply_to`、`timestamp`。所有适配器收到的消息先翻译成这个结构再进队列，出站回复也按这个结构渲染。顺序不能反——schema 不定，适配器必然各写各的。

### 2. 写两个"薄"适配器

- Telegram 侧用 long polling（内网环境省掉 webhook 证书麻烦），收到 update 后转成内部消息
- Discord 侧走 gateway 连接监听消息事件，@mention 触发规则在这一层处理

适配器只做三件事：鉴权校验、格式翻译、投递到队列。不写任何业务逻辑。

### 3. 队列解耦 + 出站渲染器

入站队列削峰；出站侧每平台一个渲染器：Telegram 用 HTML 转义，Discord 用它自己的 markdown。代码块、图片、长消息分片都在渲染器处理。Agent 核心输出的是结构化块（段落 / 代码 / 图片引用），不直接吐平台文本。

### 4. 统一会话身份

会话 key 用 `platform:user_id:chat_id` 命名空间拼接，绝不用裸 ID。这样 Agent 的记忆、工具授权、限流计数天然隔离，不会串台。

### 5. 灰度上线

先接一个测试频道跑通全链路，再放开真实群组。MCP 工具完全不用改——它们只面对 Agent 核心，根本不知道消息来自哪个平台。

## 踩坑点

- **Markdown 方言**：Telegram 的 MarkdownV2 转义规则出了名的碎，`-`、`.`、`!` 都要转；Discord 宽松得多。第一次上线代码块全是乱码，最后弃用 MarkdownV2 改用 HTML parse_mode，稳定多了。
- **长度限制不同**：Telegram 4096 字符、Discord 2000。分片逻辑必须感知代码块边界，否则长代码块会被拦腰截断。
- **限流粒度**：Discord 按 route 桶限流，Telegram 对同一 chat 约束在约 1 msg/s。出站队列必须按平台分别配速，简单 sleep 不够。
- **Thread 语义不对齐**：Discord 有 thread，Telegram 论坛群有 topic，普通群只有 reply。我的取舍是只映射两边共有的语义，其余降级为平铺消息。
- **事件不一致**：Discord 有消息编辑/删除事件，Telegram bot 拿不到已发消息的编辑回执。决定：忽略编辑类事件，只处理新增消息，避免半吊子同步。
- **webhook 安全**：如果走 webhook，Telegram 的 secret_token 和 Discord 的签名校验一个都不能省，否则等于给全网开了个无鉴权 POST 接口。

## 可复用建议

1. 先定规范消息 schema，再写适配器
2. 适配器越薄越好，所有判断放核心
3. 加一个 stdout "频道"当 dry-run，调 prompt 不用真发消息
4. 出站永远渲染结构化块，别让 Agent 直接输出平台字符串
5. 每个平台独立记延迟和失败率，坏了一个能立刻看出来

## 总结

跨平台路由的本质不是"多接一个 API"，而是把平台差异压缩到适配层，让 Agent 核心保持单一事实源。适配器写薄、身份带命名空间、出站走渲染器——这三条做到，后续再加 Slack、Matrix 基本就是照抄模板的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/9d0da319fa165398.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/72879d5e9087cfed.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/9bb909208c8493b4.png)

