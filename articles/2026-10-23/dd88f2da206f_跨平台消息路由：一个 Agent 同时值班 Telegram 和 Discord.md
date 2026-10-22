---
title: 跨平台消息路由：一个 Agent 同时值班 Telegram 和 Discord
feedId: 42333
source: 综合讨论
publishedAt: 2026-10-23
---

## 背景

我们的 Agent 最初只接了 Telegram，服务一个十来人的团队。后来社区主阵地迁到 Discord，摆面前两条路：再起一个 Agent 实例，或者给现有 Agent 加一层消息路由。再起实例意味着两份记忆、两份配置、两倍 token 开销，而且同一个问题两边答案可能不一致。我们选了后者：一个 Agent 核心，两个平台适配器。跑了一段时间，把经验整理如下。

## 问题的本质

平台差异集中在三处：

1. **收发协议**：Telegram 走 Bot API 长轮询，Discord 走 Gateway WebSocket + REST；
2. **消息格式**：Markdown 方言不同、长度上限不同（4096 vs 2000）、媒体处理路径不同；
3. **会话语义**：什么算"同一轮对话"？Telegram 群聊和 Discord 频道/线程的粒度完全不一样。

真正的难点不是"接两个 API"，而是路由层如何把异构输入归一化成 Agent 的稳定输入，再把输出还原回去。

## 做法

**1. 定义统一信封（Envelope）**。内部消息结构统一为 `platform / channel_id / user_id / thread_id / text / attachments / reply_to`。入站适配器把平台 payload 翻译成信封；出站适配器把 Agent 回复翻译回平台格式。信封是整个系统的抽象核心。

**2. 会话键设计（最关键的决策）**：

```text
session_key = f"{platform}:{channel_id}:{user_id}"
```

默认按"平台+频道+用户"切 session，避免上下文串味——Discord 群里的人不应该感知到你私聊 Telegram 时的上下文。需要跨平台身份打通时再做显式绑定，不要默认合并。

**3. Agent 核心只认信封**。核心对平台零感知，回复时带上 `reply_to`（即 session_key），由路由层决定发往何处。工具调用往往是异步的，一个跑了三分钟的查询结果回来时，路由靠 `reply_to` 找到原始出口。千万别用"最后一条收到的消息"决定回复去向，并发一多必错。

**4. 格式化下沉到适配器**。Agent 输出标准 Markdown，各适配器自行转义（Telegram MarkdownV2 的转义规则非常苛刻）、按上限切段、处理限流。

**5. 适配器独立进程**。两个适配器各自运行，通过队列与核心通信。一个平台断线重连，不影响另一个平台收发。

## 踩坑点

- **Markdown 转义**：MarkdownV2 里下划线、括号、句点都会炸，比预想的坑多得多。教训：Agent 层永远不要做平台专属转义。
- **代码块切割**：超长回复硬切 2000 字符会把代码块腰斩。按段落边界切，检测到未闭合的 ``` 要补齐。
- **session_key 只用 user_id**：重名用户或同人跨平台混流会出诡异 bug，平台前缀必须保留。
- **语音消息**：Telegram 语音是 ogg，Discord 没有对应物。在适配器层统一转文字后入信封，Agent 不感知原始格式。
- **限流**：Agent 批量回复很容易触发 Discord 429。出站必须按平台做令牌桶排队，不要天真地并发发。

## 可复用建议

1. **适配器薄、核心厚**：所有平台怪癖关在适配器里，核心代码里零 `if platform ==`；
2. **信封只加字段不改名**，保证旧适配器不崩；
3. **录真实 payload 做 fixture 回放测试**，比手写 mock 靠谱得多；
4. **接第三个平台（比如 Slack）时**，如果发现要改核心代码，说明抽象漏了——回头修抽象，不要打补丁。

## 总结

一个 Agent 服务 N 个平台，本质就是"统一信封 + 路由层 + 每平台一个薄适配器"。工程量的大头不在路由逻辑本身——那部分两天就能写完——而在格式化和边角 case 的长期打磨。把抽象边界守住，之后加平台只是加一个适配器的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/96b115e5355fd86f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/a521663343ce38a2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/2452cc94ee6c3b54.png)

