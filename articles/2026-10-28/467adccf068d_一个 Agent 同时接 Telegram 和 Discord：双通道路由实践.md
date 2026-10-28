---
title: 一个 Agent 同时接 Telegram 和 Discord：双通道路由实践
feedId: 42825
source: 综合讨论
publishedAt: 2026-10-28
---

## 背景

之前我的 OpenClaw agent 只挂在 Telegram 上，个人用没问题。但团队协作搬到了 Discord，机器人逐渐被问"为什么 Discord 上叫它没反应"。目标很朴素：**同一个 agent、同一份人格和记忆，两个平台都能正常对话**，但不想维护两个实例、两份配置。

## 问题

动手前先想清楚三件事：

1. OpenClaw 的 session 默认按 `agent:channel:chatId` 维度隔离，跨平台上下文天然不互通；
2. 两个平台的消息长度限制、格式渲染、触发方式差异很大；
3. 群聊里"什么时候该回话"的规则完全不同。

## 做法

**1. 单 agent，多通道绑定。** `openclaw.json` 里只保留一个 agent，两个通道分别配 token，用 bindings 都路由到它（字段名随版本略有差异，以 `openclaw doctor` 和文档为准）：

```json
{
  "agents": [{ "id": "main" }],
  "channels": {
    "telegram": { "botToken": "${TELEGRAM_BOT_TOKEN}" },
    "discord": { "token": "${DISCORD_BOT_TOKEN}" }
  },
  "bindings": [
    { "agentId": "main", "match": { "channel": "telegram" } },
    { "agentId": "main", "match": { "channel": "discord" } }
  ]
}
```

**2. 人格与记忆统一放 workspace。** `SOUL.md` / `AGENTS.md` 定义"你是谁"，长期上下文落 `MEMORY.md`。跨平台一致性靠 workspace 保证，而不是强行合并两个 session。

**3. 平台差异全部下沉到 channel 配置。** Discord 侧开启 `requireMention`，群里 @它才回；Telegram 侧用群白名单控制触发。长回复交给各自通道的分段配置处理，不要在 prompt 里管格式。

**4. 主动消息只走一条通道。** HEARTBEAT 类的定时提醒只在 Telegram 侧启用，避免同一个提醒 Discord 再响一遍。

## 踩坑点

- **Discord 2000 字符硬限制 + rate limit**：长回答不开分段会被静默截断，我是翻 gateway 日志才发现丢消息的。
- **Telegram MarkdownV2 转义**：让通道层做格式转换，同时要求 agent 少输出表格和深层嵌套列表，两边都稳定。
- **触发语义不一致**：Discord 需要 @提及或回复，Telegram 群默认 privacy mode 会吞掉普通消息。两边都配完，行为才"看起来一样"。
- **同一个人在两个平台是两个 session、两个身份**。别指望跨平台续聊；需要共享的结论性上下文让 agent 主动写进 `MEMORY.md`。
- **Discord 的 thread 会产生新 session**，要么接受碎片化，要么明确禁止在 thread 里触发。

## 可复用建议

- 原则记一句话：**一份人格（workspace）+ 多份通道配置 + 按 chat 隔离的 session**。
- prompt 只描述身份和边界，不描述"你在哪个平台"；平台怪癖全靠 channels 配置解决。
- `openclaw.json` 进 git，token 走环境变量；每次改路由先在测试群验证。
- 排障顺序固定：gateway 日志 → 确认 session key → 检查通道白名单和 mention 规则。大部分"没反应"都是第三步。

## 总结

单 agent 服务多通道，在 OpenClaw 里是配置问题而不是架构问题。真正的成本在平台差异和触发语义上。把差异收敛到 channel 层、尊重 session 天然隔离、记忆统一到 workspace，两个平台的体验就能基本对齐。后续如果要加第三个平台，大概率只是再写一份 channel 配置的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/e5b56f60bc4e72db.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/a7102c32e1a015ab.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/32fe28538821734c.png)

