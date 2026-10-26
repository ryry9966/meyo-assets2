---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 42619
source: 综合讨论
publishedAt: 2026-10-26
---

## 背景

我们的社区群一半在 Telegram、一半在 Discord。早期图省事，两个平台各跑一个 bot，各自带一套 prompt 和工具配置。结果就是行为漂移：改了 Telegram 侧的指令忘了 Discord，两边记忆互不相通，排查问题时还得看两份日志。这次重构的目标很明确：**一个 Agent 核心，两个“喇叭”**。

## 问题

平台差异远不止 SDK 不同：

- **消息模型**：Telegram 的 `entities` 和 Discord 的 markdown 方言、@提及写法完全是两套体系
- **长度限制**：Telegram 4096 字符，Discord 2000，切分策略不能通用
- **触达方式**：Telegram 可以长轮询，Discord 必须走 Gateway WebSocket
- **限流**：Discord 按频道 429，Telegram 按聊天约 1 msg/s
- **身份不互通**：同一个人在两边是两个 user_id

在业务代码里写 `if platform == "telegram"` 分支，两周后必然腐化。

## 做法

在 OpenClaw 里，我们把 Agent 核心跑成常驻进程，前面只挂一层薄薄的协议翻译：

1. **统一信封**：定义 `InboundMessage` schema（platform / chat_id / user_id / thread_id / text / attachments / reply_to / message_id）。两个 adapter 只做字段映射，不含任何业务判断。
2. **会话路由键**：群聊用 `platform:chat_id`，私聊用 `platform:user_id`；Discord 的 thread 和 Telegram 超级群的话题把 `thread_id` 拼进 key。Agent 的上下文与记忆按这个 key 隔离。
3. **出站 renderer**：每个平台一个格式化器，负责长文切分（按段落切，避免腰斩代码块）、转义和 mention 映射。
4. **中间加队列**：listener 收到消息写 Redis Stream，worker 消费后调 Agent。两边限流节奏不同，解耦后各自做 backoff，互不拖累。
5. **工具层共用一份**：MCP server 只挂一套，两个平台的 Agent 实例共享同样的工具和权限配置，权限收敛在一处。

## 踩坑点

- Telegram 的 MarkdownV2 转义是灾难现场，直接换 HTML parse mode，省一半调试时间。
- 别在 webhook 和 `getUpdates` 之间摇摆，会丢 offset；家用/自托管环境长轮询最稳。
- Discord 429 要老老实实读 `retry_after` 再睡，自创重试节奏只会被限得更狠。
- 两个平台的身份不要默认合并。我们踩过坑：两个不同用户恰好用了同名昵称，上下文串台。除非显式做账号绑定，记忆一律按信封里的原始 key 隔离。
- Telegram 语音是 ogg/opus，Discord 常见 mp3/m4a，进 ASR 之前先统一转码。
- 重启恢复：Discord gateway 要处理 resume，Telegram 的 `getUpdates` offset 必须持久化，否则重启后旧消息重放一轮。

## 可复用建议

- **adapter 保持笨**，所有逻辑放核心，之后接 Slack/微信只是再写一个几百行的 translator。
- **原始信封落盘**，出问题可以离线 replay，不用去翻平台后台。
- 用平台 `message_id` 做幂等去重，网络重试不会导致重复回复。
- 渠道映射全部放配置文件，禁止硬编码。
- 录制两个平台的边界 case fixture（超长文、贴纸、回复链、话题消息），跑 CI 回放，比手工点 bot 靠谱得多。

## 总结

跨平台路由的本质不是“多接一个 SDK”，而是先把消息模型收敛成一个统一信封。adapter 薄、核心厚、格式化外置、限流各自处理——这四条守住，一套 Agent 服务 N 个平台只是体力活。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/30eb3c6fa8b1ee45.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/93facbf1ec1d7fdd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/b93d86c46c716ee1.png)

