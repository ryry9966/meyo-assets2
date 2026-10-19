---
title: 一个 Agent 同时服务 Telegram 和 Discord：跨平台消息路由实践
feedId: 42024
source: 综合讨论
publishedAt: 2026-10-19
---

## 背景

最初 Agent 只挂在 Telegram 上，服务自己的小圈子。后来协作的同学基本都泡在 Discord，摆在你面前有两条路：再起一个 Agent、复制一份配置和记忆；或者让同一颗"大脑"同时接两个平台。我选了后者——OpenClaw 的架构里 channel 本来就是适配器层，Agent 和 MCP 工具才是核心，为平台差异复制业务逻辑不划算。

## 问题

双平台不是"多填一个 token"那么简单，实际要解决四件事：

1. **会话归属**：群聊里到底是谁在和 Agent 说话，上下文不能串
2. **格式差异**：两边 Markdown 方言、长度上限、媒体处理完全不同
3. **速率限制**：TG 和 Discord 的限速模型不一样，撞了会被封
4. **身份映射**：同一个人在两个平台算不算同一个用户，记忆要不要共享

## 做法

**1. 双通道接入。** 配置里声明两个 channel，token、白名单聊天/服务器、DM 策略各自独立管理：

```yaml
channels:
  telegram:
    botToken: ...
    allowedChats: [...]
  discord:
    botToken: ...
    allowedGuilds: [...]
```

**2. 统一会话键。** 默认键是 `channel:chatId`，群聊场景改成 `channel:chatId:userId`，避免群内多人共用一份上下文互相污染。私聊保持平台隔离，简单可靠。

**3. 入站归一化。** 写了一个很薄的归一化插件，把两边消息折成统一结构：文本、媒体引用、发送者、回复目标、线程 ID。Agent 只面对这个 schema，永远不知道消息从哪来。

**4. 出站适配。**
- 分块：按平台上限切（Discord 2000 / TG 4096），切分点避开代码块内部
- Markdown：Agent 输出标准 Markdown，Discord 基本直发，Telegram 转 HTML 并做转义
- 限速：出站统一过每 channel 一个令牌桶，收到 429 指数退避

**5. 观测。** 日志统一带 `channel + 消息ID` 前缀，两端问题能串成一条链路查。

## 踩坑点

- Telegram 的 MarkdownV2 转义是黑洞，`_ * ~ [ ]` 全要处理，最后直接换 HTML parse mode，省心
- 分块曾把代码块拦腰截断，Discord 上渲染全乱。切分前先扫描未闭合的 ``` 围栏
- 群聊最初按群建会话，两个人的对话互相出现在对方上下文里，改成 per-user 才正常
- 同一个人双平台会出现两份记忆。目前用手工绑定的身份映射表，没做自动合并——隐私上宁可保守
- 长任务执行时两边的"正在输入"状态要周期性刷新，否则用户以为 Agent 挂了

## 可复用建议

- Channel 保持薄：只做收发和格式转换，不要塞业务逻辑
- 限速放在出站队列，别让 Agent 感知平台差异
- 消息 ID 全链路透传，排障效率差一个量级
- 先把一个平台跑稳再接第二个；归一化 schema 定型后，后续加 Slack 之类就是复制一份适配器
- MCP 工具保持 channel 无关，平台问题全部在路由层解决

## 总结

一套 Agent 双平台，核心工作量不在 Agent 本身，而在那层不到三百行的归一化 + 适配代码。OpenClaw 这种"网关 + 适配器"的结构天然适合这种需求，真正要想清楚的只有两件事：会话粒度和身份映射策略。剩下的都是体力活。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/d8e881e495327d05.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/79a86e565bdef6d7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/5dabb9ce23d07901.png)

