---
title: 一个 Agent，两个入口：Telegram + Discord 跨平台路由实战
feedId: 40941
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景

场景很典型：开发协作和社区讨论在 Discord，移动端快问快答在 Telegram。最早我们跑了两套独立 Agent，很快暴露问题——两边上下文互不相通，同一件事要重复交代两遍，工具配置改一处漏一处。这篇帖子记录把两个平台收敛到同一个 OpenClaw Agent 核心的过程，重点在路由设计。

## 问题拆开是三件事

1. **接入层**：两个平台的凭据、事件模型、长连接方式完全不同；
2. **会话路由**：消息进来后归到哪个 session？全平台共享一个上下文，还是按聊天隔离？
3. **输出适配**：长度上限（Discord 2000 字符 / Telegram 4096）、Markdown 方言、附件大小都不一样。

## 做法

**第一步，凭据与通道。** Telegram 找 BotFather 拿 token；Discord 在开发者后台建应用，务必开启 Message Content Intent，否则 bot 在线但读不到消息。两个 channel 在 gateway 里并行启用，指向同一个 agent。

配置示意（字段名以当前版本为准，表达的是结构思路）：

```jsonc
{
  "agents": [{ "id": "main" }],
  "channels": {
    "telegram": { "botToken": "***" },
    "discord":  { "token": "***" }
  },
  "routing": {
    "default": "per-chat",
    "bindings": [
      { "channel": "telegram", "chatId": "-1001234...", "session": "ops-core" },
      { "channel": "discord",  "channelId": "98765...",  "session": "ops-core" }
    ]
  }
}
```

**第二步，路由策略。** 默认按会话隔离（sessionKey 形如 `platform:chatId`），避免陌生群和私聊互相污染上下文。需要共享记忆的场景——比如两边各有一个“运维频道”——用显式 binding 绑到同一个 session。别一上来就全局共享，回滚成本很高。

**第三步，输出统一。** 在系统提示里注入消息来源平台，让 Agent 知道对面是谁；同时约束它输出“保守 Markdown”：不用表格、少用嵌套列表、代码块优先。超长回复交给 adapter 按平台上限切片。

**第四步，工具层走 MCP。** 平台差异全部拦在 adapter，工具保持平台无关。Agent 不管消息从哪来，只调用同一套 MCP 工具。

## 踩坑点

- **Discord Intent 没开**：最经典的“上线即失聪”。排查顺序：在线状态 → Intent → 频道权限 → allowlist。
- **桥接回环**：群里如果同时跑着跨平台转发机器人，Agent 会把转发消息当用户输入，回复自己。忽略名单里必须排除 bridge 的 ID。
- **身份双写**：同一个人在两个平台是两个 ID，任何按用户做权限或记忆绑定的工具都要维护映射表，否则出现“他那边认识我，这边不认识我”。
- **长连接选择**：内网机器 Telegram 用 long polling 就够；webhook 需要公网 + TLS，不为一朵花折腾。
- **渲染差异**：两个平台的 Markdown 子集行为不同，富文本实测持续翻车，最后统一降级到最朴素的格式才稳定。

## 可复用建议

1. 把 routing 配置当代码管理，进仓库、走 review；
2. 日志每轮带 `platform/chat` 标签，跨平台问题才查得动；
3. 先在低风险频道灰度一周，确认会话隔离符合预期再扩；
4. 两个平台的 allowlist 策略要保持一致并定期审计，否则边界规则会被“另一边”绕开。

## 总结

跨平台既不是把 token 复制两份那么简单，也不是把 Agent 跑两份那么浪费。核心就两件事：**adapter 层吃掉所有平台差异，routing 层明确回答“上下文归谁”**。做到这两点，之后接入第三、第四个平台，只是配置问题。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/ef09cbd3f515b7c3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/2061c008ceb54a2e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/c0650be93c784cc7.png)

