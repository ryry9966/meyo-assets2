---
title: OpenClaw 跨平台消息路由：一个 Agent 同时值守 Telegram 和 Discord
feedId: 41364
source: 综合讨论
publishedAt: 2026-10-13
---

## 背景

需求很朴素：我在 Telegram 和 Discord 各有一批用户，但不想维护两套 bot 逻辑。OpenClaw 的架构里 agent 本身是平台无关的，channel 只是收发适配层，MCP 工具和插件挂在 agent 上，接几个平台就共享几个平台的工具。所以目标不是"两个机器人"，而是**一个大脑，两个门口**。

## 问题

真正要解决的其实只有三件事：

1. **会话归属**：同一个用户从两个平台进来，上下文要不要混用？
2. **格式差异**：Markdown 方言、长度上限（Discord 2000 / Telegram 4096）、附件语义都不一样。
3. **节奏差异**：两边限流策略不同，轮询和推送的事件形态也不同。

## 做法

**第一步：双 channel 接入，session 按来源隔离。**

```json5
{
  channels: {
    telegram: { botToken: "…", allowedChats: ["-100123…"] },
    discord: { botToken: "…", intents: ["GuildMessages", "MessageContent"] }
  },
  routing: { default: "agent://main" }
}
```

sessionKey 用 `channel:chatId`，两个平台天然隔离，不会串上下文。需要共享的（长期记忆、工具状态）放 workspace 层，不放 session 层。

**第二步：出口渲染下沉。** agent 只输出"语义消息"——段落数组加附件引用，由各 channel adapter 负责转成 Telegram HTML 或 Discord Markdown，超过阈值按段落切片。这是整套方案里最值钱的一条。

**第三步：路由旁路。** 公告类频道设为只读：消息照常进 agent 做记录，但不回帖，避免 agent 在不合适的场合抢话。

**第四步：出口排队。** 每个 channel 一个发送队列，按平台限速走，429 一律指数退避，不做额外花活。

## 踩坑点

- **Discord bot 在线但永远沉默**：MESSAGE CONTENT Intent 没开，收到的消息 body 是空的。新号最高频的坑。
- **Telegram MarkdownV2 转义地狱**：回复里带下划线、点号就解析失败，直接切 HTML parse mode，一劳永逸。
- **切片阈值写死**：2000 和 4096 是两回事，阈值必须配置在 channel 层，别让 agent 关心。
- **编辑/撤回语义不对称**：Telegram 有 edited_message，Discord 的 update 事件不可靠，别做"改答案"类功能假设。
- **本机部署别急着上 webhook**：没有固定 HTTPS 域名就用 long polling，够用且省心。
- **双平台同时触发同一工具**：workspace 文件并发写冲突，agent 侧加串行队列或文件锁。
- **唤醒词和 @ 提及打架**：两个平台提及格式不同，统一在 adapter 层归一化成"被召唤"事件。

## 可复用建议

1. 平台差异全部下沉到 channel 层，agent 永远输出语义消息——以后加 Slack 只是再写一个 adapter。
2. 每个平台留一个测试群 + dry-run 开关，改动先跑 dry-run 再上线。
3. 日志强制带 `channel` 和 `chatId` 标签，跨平台排障基本全靠它。
4. 唤醒白名单做成配置而不是写进系统提示词，改权限不用重启 agent。

## 总结

一个 Agent 服务两个平台，工程上没有黑魔法：**session 归属清晰、渲染下沉到 adapter、出口统一排队**。做完这三件事，跨平台对 agent 来说只是"多了一根进水管"。排障时先看日志标签定位平台，再查 Intent、parse mode 这类平台侧配置——九成问题出在边界层，而不是 agent 本身。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/5c3fa42ebff7fc12.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/0ed8276d9365db39.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/7e51dce216c85ece.png)

