---
title: 一个 Agent 同时服务 Telegram 和 Discord：跨平台消息路由实践
feedId: 38950
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

在 OpenClaw 里把 Agent 核心跑通后，现实问题马上来了：团队一部分人在 Telegram，一部分在 Discord。最初的做法是各养一个 bot、各挂一份记忆，结果同一个人在两边面对两个"人格"，工具调用记录分裂，MCP 工具重复配置，维护成本直接翻倍。这篇记录一次收敛：一个 Agent 核心，两个平台入口。

## 问题

拆开看是四个问题：

1. **会话归属**：同一用户跨平台时，session 怎么算？
2. **消息模型**：Telegram 的 MarkdownV2、4096 字符上限，Discord 的 2000 字符、embed、slash command，差异不小。
3. **速率限制**：两个平台限流策略完全不同，直接裸发会被 429。
4. **运维形态**：Telegram 可以 long polling，Discord 必须 gateway websocket，进程模型要先想清楚。

## 做法

核心思路一句话：**adapter 尽量薄，核心保持平台无关。**

1. **定义统一信封**。所有入站消息归一化：

```json
{
  "platform": "telegram|discord",
  "chat_id": "...",
  "user_id": "...",
  "text": "...",
  "attachments": [],
  "msg_id": "..."
}
```

2. **两个 adapter 各做成一个渠道插件**，只做三件事：normalize（入站）、render（出站）、ack（消息去重）。Agent 核心不 import 任何平台 SDK。

3. **路由层用 `platform:user_id` 生成 session key**。第一版建议就按平台隔离，不要急着做跨平台身份合并——那是产品问题，不是工程问题，先跑起来再说。

4. **出站走每平台一个队列**。渲染器把核心输出的中性 Markdown 翻译成平台方言；超长输出降级为文件发送（Telegram sendDocument / Discord attachment），不要靠分片硬切，分片切在代码块中间非常难看。

5. **MCP 工具层完全共享**，与渠道无关。部署上是单进程多 adapter，session store 用 SQLite 加写锁，当前规模没必要上 Redis。

## 踩坑点

- **Markdown 方言**：Discord 和 Telegram 的转义规则几乎不兼容，直接透传核心的 Markdown 必炸。务必做 per-platform renderer，且都留纯文本 fallback。
- **命令漂移**：`/reset` 这类命令要在 BotFather 和 Discord Developer Portal 各注册一次，行为很快就不同步。解法是命令解析放核心，平台只透传文本。
- **限流**：Telegram 单 chat 约 1 msg/s，Discord 每 channel 更严。广播类操作必须过队列加指数退避，429 重试时注意读 Retry-After。
- **事件重复**：websocket 重连后 Discord 会重放事件，msg_id 去重要在 ack 层做掉，否则用户收到重复回复。
- **长任务反馈**：工具调用超过 10 秒时，先在两个平台各发一条"处理中"，体验差异比想象中大。

## 可复用建议

- envelope 里带 `schema_version`，后面接第三个平台（比如 Slack）不用改核心。
- 按渠道打标记录延迟和 token 消耗，两周后你会用得上——两边用户的行为模式差异非常明显。
- 守住"adapter 不写业务逻辑"这条线，加新平台就是一天的体力活；破了这条线，每个新平台都是一次重构。

## 总结

跨平台路由本身不难，难的是忍住不在 adapter 里塞逻辑。核心平台无关、出站渲染分层、限流交给队列，这三件事做完，一个 Agent 服务 N 个渠道就是复制粘贴的工作量。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/0a3568d80c2544a0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/4a9b2333cade158a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/ca332b79b8c4b270.png)

