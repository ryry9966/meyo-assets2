---
title: 一个 Agent 服务两端：Telegram + Discord 的消息路由实践
feedId: 39923
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

社区里的用户一部分在 Telegram，一部分在 Discord。之前我们的做法是各起一个 bot，各自接一套 Agent。跑了一段时间问题很明显：同一个用户在两个平台问，Agent 表现得像失忆；MCP 工具要挂两遍配置；改一次 prompt 要重启两个进程。这篇文章记录我们把它收敛成「一套核心 + 两个适配器」的过程。

## 问题

直觉方案是两个进程各自独立跑，但真正麻烦的不是「接两个平台」，而是平台差异会渗进 Agent 核心里：

- Markdown 方言不同（Telegram 的 MarkdownV2 转义和 Discord 完全不是一回事）
- 长度限制不同（4096 vs 2000），分片策略没法共用
- 限流机制不同，一个平台 429 能把整个队列堵死
- 会话归属不清：按用户隔离还是按群隔离？

结论：必须把「平台差异」收敛到边缘，核心层只认一种消息格式。

## 做法

整体分三层：

**1. 适配层（每平台一个 adapter）**
只负责收发、鉴权、限流重试，把消息归一化成统一 envelope：

```json
{
  "platform": "telegram|discord",
  "channel_id": "...",
  "thread_id": null,
  "user_id": "...",
  "msg_id": "...",
  "content": "...",
  "attachments": [],
  "reply_to": null
}
```

**2. 路由层**
根据 envelope 生成 session key：
- 群组/频道：`platform + channel_id`（有 thread 再拼 thread_id）
- 私聊：`platform + user_id`

跨平台记忆单独决策：我们选择 session 隔离、长期记忆共享，避免群里串台的同时保住人格和上下文连续性。

**3. Agent 核心 + MCP 工具**
完全不感知平台。回复也是统一结构，由适配层反向渲染：格式转换 → 按平台限制分片 → 限流退避后发送。

## 踩坑点

- **session key 一开始用了 user_id**，群里所有人共享一个会话，当场串台。群聊必须按 channel 维度隔离。
- **必须过滤 bot 自己的消息和回声**，否则两个 agent 对接或消息转发场景会进入无限循环，日志刷到磁盘爆炸。
- **Markdown 别在核心层拼**。Telegram MarkdownV2 有十几个转义字符，渲染失败宁可降级纯文本，也不要让消息发不出去。
- **附件传 URL 不传本地路径**，核心层拿可下载链接再交给工具处理，两个平台的媒体语义差异留在 adapter 里消化。
- **429 处理放在 adapter**：Discord 按桶限流，Telegram 全局 + 按 chat，退避策略别共用一把全局锁。
- 同一个 token 别起两个实例同时 polling；webhook 和 polling 二选一，这是我们最蠢的一次事故。

## 可复用建议

- adapter 保持「薄」：只做协议转换，不写任何业务逻辑
- envelope 加 `version` 字段，之后接 Slack、QQ 也不用动核心
- 发送侧做幂等：`platform + msg_id` 作去重键，重试不会重复播报
- 上线前用 dry-run 跑一天，把两个平台的归一化结果各 dump 一份，检查字段覆盖率
- 日志带 envelope id，排障时能串起「收到 → 处理 → 发出」全链路

## 总结

跨平台路由的本质不是多写一个 bot，而是设计一层足够小的 envelope，把平台语义全部挡在边缘。做到这一点后，加新平台只是多写一个 adapter，Agent 核心、MCP 工具、记忆和 prompt 全部复用。目前这套结构稳定跑了两周，日均消息量不大但链路日志干净，排障时间明显下降。欢迎在社区里交流你们的 session key 设计和限流策略。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/171730b06a9b3d55.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/3deebd7ff9663305.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/c16a65304ae2ef1c.png)

