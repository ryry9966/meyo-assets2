---
title: 一套大脑、两个入口：一个 Agent 同时服务 Telegram 与 Discord 的路由实践
feedId: 41649
source: 综合讨论
publishedAt: 2026-10-16
---

## 背景

我们的场景挺典型：技术讨论在 Discord，外部协作者和移动场景在 Telegram。之前是两个 bot 各跑各的，结果同一套知识库配了两份提示词——Agent 在 Discord 里记得的事，到 Telegram 一问三不知。这次改造目标很朴素：**一个 Agent、一套记忆和工具、两个消息入口**。

## 问题

跨平台不是"多接一个渠道"那么简单，实际要回答三个问题：

1. **路由**：两个渠道的消息怎么进同一个 Agent，回复又怎么原路返回，不能串门；
2. **会话边界**：两边上下文是隔离还是共享？隔离则记忆割裂，共享则可能"在 TG 里聊到 Discord 群的内容"；
3. **表现层**：两边消息格式、长度上限、群权限差异很大。

## 做法

核心思路是用 bindings 把多个 channel 指向同一个 agent，而不是复制一份 Agent 配置：

```json
{
  "agents": {
    "list": [{ "id": "main", "workspace": "~/.openclaw/workspace" }]
  },
  "channels": {
    "telegram": { "botToken": "${TELEGRAM_BOT_TOKEN}" },
    "discord":  { "token": "${DISCORD_BOT_TOKEN}" }
  },
  "bindings": [
    { "agentId": "main", "match": { "channel": "telegram" } },
    { "agentId": "main", "match": { "channel": "discord" } }
  ]
}
```

步骤：

1. **先把 Telegram 单渠道跑稳**，确认 gateway 正常、回复回路通；
2. **接 Discord**：建 Application + Bot 拿到 token，务必在开发者后台开启 **Message Content Intent**；
3. **写 bindings**：两个 match 都指向 `main`，无需重复定义 agent；
4. **定会话策略**：默认按渠道 + 会话对象隔离，适合多用户；如果是个人助理想跨平台连续记忆，再考虑共享主会话，但要接受上下文互通的代价；
5. **改提示词**：明确要求 Agent 输出平台中立的纯文本，不假设渲染环境。

改完重启 gateway，`openclaw doctor` 过一遍，再从两边各发一条消息，看日志里 inbound 的走向。

## 踩坑点

- **Discord 静默不回**：九成是 Message Content Intent 没开，而且这种失败在日志里往往不明显，先查这个；
- **2000 字符上限**：长回复直接被拒。让 Agent 分段输出，或在发送层做切分兜底；
- **Telegram MarkdownV2 转义**：下划线、反引号很容易炸格式，实践是默认纯文本，确有需要再开；
- **群隐私模式**：Telegram bot 在群里默认收不到普通消息，要么关 privacy mode，要么让人 @ 它；
- **会话泄漏**：共享会话后，A 平台的对话可能出现在 B 平台的回复语境里，共享前想清楚；
- **Discord 频率限制**：短时间多条回复会撞限速，批量消息要合并或排队；
- **热改配置不生效**：channels/bindings 改动记得重启 gateway 再验证。

## 可复用建议

- **一 Agent 多渠道是最小可行架构**，先别急着上多 Agent。bindings 是表驱动的，之后接 Slack、WhatsApp 只是再加一行；
- **token 全走环境变量**，配置文件里不落密钥；
- **提示词里写死"平台中立"**，比事后修格式问题省得多；
- **排查顺序固定**：先确认消息进没进来，再查路由命中，最后查发送层——别一上来就怀疑模型；
- **按 peer 维度测**：私聊、群聊、@提及 各走一遍，覆盖完整的 reply 回路。

## 总结

这次改造代码量接近零，收益却是记忆统一、维护减半。经验浓缩成一句话：**路由和会话边界是跨平台的核心，格式差异交给发送层兜底**。先跑通单渠道，再用 bindings 复制入口，比一步到位稳得多。大家在多渠道路由上踩过别的坑，欢迎跟帖交流。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/5bf2ab7d18c3dce1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/0430323ca0f19ca6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/f1452d27a7e732e1.png)

