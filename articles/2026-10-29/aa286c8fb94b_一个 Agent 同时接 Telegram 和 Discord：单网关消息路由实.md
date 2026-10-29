---
title: 一个 Agent 同时接 Telegram 和 Discord：单网关消息路由实践
feedId: 42896
source: 综合讨论
publishedAt: 2026-10-29
---

## 背景

我的使用场景很典型：技术社区在 Discord，家人朋友和手机端的快捷交互在 Telegram。最初图省事跑了两套 agent——两个 gateway、两份 workspace。两周后就出现漂移：人格描述不一致、记忆各记各的、模型配置改了一处漏一处。目标于是变成：单 gateway、单 agent、同一份工作区，同时服务两个平台，但上下文按会话隔离。

## 问题

拆开看，要解决的是四件事：

- **会话模型差异**：Discord 有频道和子区（thread），Telegram 有群组和话题（topics），session key 的粒度必须对齐；
- **路由边界**：哪些聊天共享一个会话、哪些严格分开；
- **出站差异**：Markdown 方言、长消息分片规则两边不一样；
- **安全与投递**：allowlist 防陌生人，heartbeat/定时任务的主动消息得知道往哪儿发。

## 做法

1. **在 `openclaw.json` 里配两个 channel**（字段名以你版本的配置校验为准）：

```json
{
  "channels": {
    "telegram": {
      "botToken": "${TELEGRAM_BOT_TOKEN}",
      "dmPolicy": "pairing",
      "allowFrom": ["<你的 tg user id>"],
      "groups": { "<群 id>": { "requireMention": true } }
    },
    "discord": {
      "token": "${DISCORD_BOT_TOKEN}",
      "dmPolicy": "pairing",
      "guilds": {
        "<guild id>": {
          "channels": { "<channel id>": { "requireMention": false } }
        }
      }
    }
  }
}
```

2. **单 agent 默认承接全部路由**。所有聊天默认落到 default agent，一份 workspace、一份人格、一套工具。以后如果要拆"工作/生活"两个 agent，用 bindings 按群/guild 绑定即可，不用重建 gateway。
3. **会话粒度用默认的"渠道 + 会话 ID"键**（形如 `telegram:group:xxx`、`discord:channel:xxx`），确保 Discord 子区和 Telegram 话题各自有独立 key。拿不准当前落在哪个 session，就发 `/status` 看。
4. **统一跨平台身份**：在 workspace 记忆里写明"Discord 的 A 和 Telegram 的 B 是同一人"，并要求 agent 遇到歧义先问，不猜。
5. **验证**：`openclaw gateway --verbose` 看路由日志，两边各发一条引用回复，确认 reply 链路和 session 归属。

## 踩坑点

1. **线程级 session 没开**：Discord forum 的帖子全挤进频道级上下文，token 很快爆掉。线程/话题务必有独立 key。
2. **格式翻车**：Discord 吃 Markdown，Telegram 出站更挑剔，长文分片规则也不同。让 agent 输出朴素格式，或在出站层按渠道转换，别指望模型自己适配。
3. **主动消息串台**：heartbeat 默认投给"最近活跃"的会话，两个平台同时在聊就投错。定时任务和 heartbeat 要显式指定投递渠道。
4. **限流**：Discord 按频道限速，Telegram 按聊天限速，批量播报自己加节流，别依赖重试。
5. **allowlist 写错**的典型症状是"群里机器人对谁都答"。先收紧到自己的 ID，验证后再逐步放开；两个 token 都走环境变量，不进配置文件。

## 可复用建议

- 渠道适配层保持薄：平台相关的脏活（格式、分片、限流）全在适配层消化，模型不感知平台差异；
- 把 session key 当作路由契约，改配置前后都用 `/status` 确认；
- 配置进 git，diff 可回滚；
- 需要 agent 分工时用 bindings，永远不要复制第二套 gateway。

## 总结

单 agent 双平台完全可行，核心就三点：**会话粒度选对、跨平台身份写清楚、主动消息显式投递**。剩下的渲染和限流差异都在出站层解决。跑通之后，两个平台对 agent 来说只是两个前端，维护成本回到一份 workspace 上，这才是一个人维护得动的形态。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/32db73f97df85b65.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/4a111fa3c420e722.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/99ab7809324c72bb.png)

