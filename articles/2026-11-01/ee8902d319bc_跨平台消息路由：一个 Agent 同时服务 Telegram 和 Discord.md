---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 43105
source: 综合讨论
publishedAt: 2026-11-01
---

## 背景

我在本地跑了一个 OpenClaw Agent，最初只接 Telegram，自己用着顺手。后来团队和群友都聚集在 Discord 上，再部署第二套 Agent 显然不划算：模型配置要维护两份、上下文互相割裂、MCP 工具还得同步。目标很朴素：一个 Agent 实例当"大脑"，Telegram 和 Discord 只是两扇不同的"门"。

## 问题

OpenClaw 支持多 channel 并存，但"能跑通"和"跑得干净"是两回事。直接把两个 channel 配上，会遇到几类典型问题：

- 会话串扰：不同平台、不同聊天的上下文混在一起；
- 触发不一致：Telegram 里 @机器人 能回，Discord 群里却不响应；
- 格式错乱：Discord 的 Markdown 和 Telegram 的 parse mode 不是同一个方言；
- 媒体形态不同：一边是 CDN 链接，一边是文件下载接口。

## 做法

**1. 准备凭证。** Telegram 走 BotFather 拿 bot token；Discord 在开发者后台建应用拿 token，并且务必打开 MESSAGE CONTENT 特权 intent，否则机器人上线了也不读消息。

**2. 双 channel 配置。** 在 `~/.openclaw/openclaw.json` 里并列声明：

```json
{
  "channels": {
    "telegram": { "botToken": "..." },
    "discord":  { "token": "..." }
  }
}
```

重启 gateway，日志确认两个 adapter 都连上，分别私聊发一句话验证。

**3. 会话与路由策略。** 关键认知：session key 默认按"平台 + 聊天"隔离，这是对的，别为了"同一个 Agent"强行合并会话。要统一的是人格和工具层——system prompt、MCP server、插件都挂在 Agent 层，channel 层只管收发。群聊用 allowlist 控制触发范围，DM 按 pairing 绑定，避免陌生人白嫖 token。

**4. 行为对齐。** 在两个平台各发一组测试消息（长短文、代码块、图片、语音），对比回复质量。格式问题在 Agent 层约定：让模型输出朴素 Markdown，由各 channel 适配层做方言转换，而不是让模型针对平台写不同格式。

## 踩坑点

- **Discord 静默**：intent 没开是最常见的坑，表现为在线但不回，日志里甚至没报错。
- **Telegram 断流**：早前测试设过 webhook 没清掉，和 polling 冲突，删掉即恢复。
- **主动通知发两份**：gateway 的心跳/通知若不指定目标，会向两个平台都推，通知路由要显式写进配置。
- **媒体链接时效**：Discord 附件 URL 有时效，Agent 把链接记进长期记忆后过几天就失效。需要留档的媒体，先落本地再引用路径。
- **群聊触发差异**：Telegram 群默认要求 @提及，Discord 群还要确认机器人有读历史权限，两边策略分开配。

## 可复用建议

1. **传输与大脑分层**：人格、记忆、工具永远在 Agent 层，channel 只做协议适配。以后接 Slack、WhatsApp 只需新增 channel 配置。
2. **维护一张路由表**：哪些聊天 → 哪个 Agent、会话策略、触发条件，写进配置注释，出问题先查表。
3. **配置进 Git**：双 channel 之后 openclaw.json 的改动频率翻倍，漂移是真实发生的。
4. **日志按 channel 打标**：先观察再扩展，别一上来接五个平台。

## 总结

多平台接入的难点从来不是"连上"，而是会话隔离、路由显式化和格式适配这三件脏活。把 Agent 当大脑、channel 当门，边界划清后，一个实例服务 N 个平台的边际成本其实很低。下一步我打算把通知路由抽成独立的规则文件，替代散落在配置里的硬编码。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/f9d7e7eeda506c38.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/bf1c8640e3cfb858.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/faa927d4fbcda5ef.png)

