---
title: 跨平台消息路由：一个 Agent 同时守住 Telegram 和 Discord
feedId: 42832
source: 综合讨论
publishedAt: 2026-10-28
---

## 背景

我们团队日常沟通一半在 Telegram（对外、移动端），一半在 Discord（社区和开发协作）。之前接 OpenClaw 时只开了 Telegram 渠道，Discord 侧的同学每次都得切回来问一遍同样的问题。重复回答事小，上下文割裂事大——Agent 在两边各有一套记忆，状态不一致，时间长了像两个不同的人格。

目标很朴素：一个 Agent 实例、一份配置、一套记忆，同时响应两个渠道。

## 问题

拆开看其实是四个子问题：

1. **接入**：两个渠道的鉴权、心跳、断线重连逻辑完全不同；
2. **路由**：消息进来后归到哪个 agent、哪条 session，规则要显式，不能靠猜；
3. **会话**：同一个人在两边是两个 ID，session 要不要合并？合并了怕串台；
4. **输出**：两个平台的 Markdown 方言、长度限制、回复引用机制都不一样，直接转发会出现排版事故。

## 做法

核心思路一句话：**Agent 保持平台无关，所有平台差异收敛在 channel 层。**

第一步，在 gateway 配置里同时启用两个渠道（token 分别来自 BotFather 和 Discord Developer Portal，后者记得开 Message Content Intent），并把两个渠道都绑定到同一个 agent。示例（字段名以你本地版本为准）：

```json
{
  "channels": {
    "telegram": { "botToken": "..." },
    "discord": { "token": "..." }
  },
  "bindings": [
    { "agentId": "main", "match": { "channel": "telegram" } },
    { "agentId": "main", "match": { "channel": "discord" } }
  ]
}
```

第二步，明确触发规则：Telegram 私聊直连、群聊必须 @；Discord 只响应 mention。宁可少响应，不要抢戏。

第三步，session 策略。默认按 channel + chat 隔离，最稳。我们试过把同一用户的 tg id 和 discord id 映射到同一条 session，私聊场景体验不错，但多人群聊下串台风险高，最后只对私聊开放。

第四步，输出适配。system prompt 里只约束“输出平台中性的 Markdown”，换行、表格降级、长文截断全部交给 channel 层处理。Discord 单条 2000 字符，超长自动分段并附摘要。

第五步，工具和记忆统一走 MCP / skill。两个渠道共享同一个工具面，这才叫得上一句“同一个 Agent"。

## 踩坑点

- **Discord 收不到消息正文**：十有八九是 Message Content Intent 没开，而且 Bot 页和权限页要同时勾上；
- **Telegram webhook 内网穿透不稳**：本机跑 gateway 先用 long polling，别一上来就 webhook；
- **格式转义地狱**：MarkdownV2 的转义比 Discord 严格得多，别指望 agent 自己处理对，交给 channel 层；
- **重复回复**：网络抖动触发重试，两个渠道各回了一次。给回复加幂等键（按 message id 去重）；
- **附件链接过期**：Discord 附件 URL 有时效，先转存再引用，别把原始链接直接塞给 agent。

## 可复用建议

- 路由、触发、session 规则全部放配置，不写进 prompt；
- 每个渠道只写一层薄适配，agent 代码里不出现 `if (platform)`；
- 日志统一打上 `channel:chat_id` 标签，排障效率翻倍；
- 先私聊灰度一周，再放开群聊；群聊务必加白名单，防止被拉进陌生群刷爆 token。

## 总结

多渠道接入的难点从来不是“再接一个平台”，而是路由和会话这两个语义问题。把平台差异关在 channel 层、把路由规则写成显式配置、对 session 合并保持保守，一个 Agent 服务多个渠道就是体力活而不是玄学。这套结构换成 Slack、QQ 或其他渠道，同样成立。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/ead5d85f32ff5c14.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/0c0d03e4e9221ef4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/603664767a3b28f0.png)

