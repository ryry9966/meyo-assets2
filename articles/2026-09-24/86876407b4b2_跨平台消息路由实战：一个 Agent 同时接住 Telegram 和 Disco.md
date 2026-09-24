---
title: 跨平台消息路由实战：一个 Agent 同时接住 Telegram 和 Discord
feedId: 38744
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

我最初的做法很典型：Telegram 上一个私聊 bot，Discord 服务器里另一个 bot，两边各跑一套 agent。prompt 复制两份，memory 各自为政，改一次人设要手动同步两个工作区，纯人肉双写。目标于是很朴素：**一个 agent（同一 workspace、同一份 memory、同一套 skills），对外暴露两个消息入口**。

## 问题

OpenClaw 默认就支持多个 channel 共用一个 agent，难点不在“能不能接”，而在三处平台差异：

- **会话边界**：不能把两个平台塞进同一个 session，上下文会互相污染，还涉及隐私；
- **触发规则**：Discord 群频道靠 @mention，Telegram 群还有 privacy mode，不显式配置就会出现“一边活、一边哑”；
- **消息格式**：两边 markdown 子集不同，表格、长代码块、长回复分片表现不一致。

## 做法

1. **一个 agent，两个 channel**。核心配置大致是：

```json
{
  "agents": [{ "id": "main", "workspace": "~/openclaw/workspace" }],
  "channels": {
    "telegram": { "botToken": "...", "dmPolicy": "paired", "groupPolicy": "allowlist" },
    "discord": { "token": "...", "guilds": { "<guildId>": { "users": ["<myId>"], "requireMention": true } } }
  }
}
```

2. **会话作用域交给默认行为**：session key 按 channel + 会话隔离，两个平台的短上下文天然分开；长期记忆统一放在 workspace 的 memory 文件里。agent 是同一个，人格和知识自然跨平台一致。
3. **触发规则显式声明**：Discord guild 频道 `requireMention`；Telegram 群走 allowlist + @提及。DM 一律 pairing/allowlist，别做 open relay。
4. **格式做减法**：让 agent 输出以普通段落和短代码块为主，少用表格；长回复交给分片处理。
5. **观测**：调试期在日志里确认每条消息命中的 channel 和 session key，`openclaw doctor` 顺手跑一遍。

## 踩坑点

1. **Discord 的 MESSAGE CONTENT intent 没开**：私聊正常、群频道收不到正文，表现为“已读不回”。开发者后台勾上后重启 gateway。
2. **Telegram 群 privacy mode**：默认只收到命令和回复类消息。去 BotFather 关掉 privacy，或把 bot 设为群管理员。
3. **分片与限速**：长回复在 Discord 撞限速比 Telegram 明显，分片阈值调小；分片可能打断代码块渲染，长代码收敛到文件或链接。
4. **媒体差异**：两边附件上限都偏小且不同，语音格式也不同（Telegram 偏好 OGG/Opus）。跨平台媒体让 agent 用工具落盘后再发链接，比直接转发稳。
5. **不要为了“共享记忆”手动把两个 session 指到同一个 key**——上下文会串，隐私不可控。要共享的是 memory 文件和 skills，不是 session。
6. **heartbeat / cron 是 agent 级的**：别按平台直觉复制定时任务，否则同一件事会推两遍。

## 可复用建议

- **channel 薄、agent 厚**：平台差异尽量在 channel 配置层消化（mention、allowlist、分片），不要往 prompt 里塞平台判断逻辑。
- prompt 写成平台无关版本，人格与知识统一放 workspace，输出格式约束全局从简。
- 每接一个新平台，跑三条冒烟消息：私聊、群里被 @、群里没人理它（确认它不抢答）。
- allowlist 从第一天就配上，不要裸奔后再补。

## 总结

跨平台路由不是把 token 填两份就完事，真正的工作量集中在会话边界、触发规则、格式适配这三处。一句话复述结论：**同一个 agent，多个 session，共享 memory，差异留在 channel 层**。后来加新的消息入口时，这套结构基本照搬，改的只有 channel 配置。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/8f15949b7e7fc4e1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/ebf34be88d9410dc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/6b6461b20b15f60e.png)

