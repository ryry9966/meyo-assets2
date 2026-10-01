---
title: 一个 Agent 吃两路消息：OpenClaw 同时接 Telegram 和 Discord 的路由实践
feedId: 39964
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

我的 OpenClaw 网关原本只挂了 Telegram，干的是个人提醒、随手问答这类活。后来发现社区里不少人只活跃在 Discord，再起一套 Agent 意味着两份记忆、两份配置、两条维护线。目标很朴素：**同一个大脑、同一份记忆、同一组 MCP 工具，前面挂两个通道。**

## 问题

OpenClaw 的架构里 channel 本来就是薄适配层，“一 Agent 多通道”官方支持，但真正的工作量全在策略层：

1. **会话边界**：TG 私聊和 Discord 频道要不要共享上下文？实测合并 session 后两边话题互相污染，A 平台的测试数据会漏进 B 平台的回答里。
2. **身份映射**：同一个人在两个平台是两个 ID，记忆检索和 allowlist 都对不上。
3. **格式与限制**：Telegram 4096 字符 + MarkdownV2 转义地狱，Discord 2000 字符 + 自家 Markdown 方言。
4. **主动消息路由**：heartbeat、定时任务的结果到底投给谁？

## 做法

1. **通道接入**：BotFather 拿 Telegram token；Discord Developer Portal 建应用，**务必打开 MESSAGE CONTENT intent**。配置里分别填 token，allowlist 先收紧到自己的账号。
2. **会话隔离**：保持默认的 per-chat session key，不要为了“跨平台连续性”去合并 session。需要连续性时，把关键结论写进 memory 文件，而不是共享对话历史。
3. **触发策略**：Discord 群组频道设 mention-only，两边私聊全量响应；只读频道干脆不挂，从源头避免 bot 在自己的消息上再触发一轮。
4. **格式中转**：Agent 输出统一定为平台无关的 Markdown，转义、按段落分块到各自长度上限、embed 化，全部下沉到通道适配层。prompt 和 skill 里永远不写平台方言。
5. **主动消息**：定一张路由表——heartbeat 和告警默认投递到 TG 私聊，社区问答归 Discord 指定频道。路由表放配置里，别写死在代码。

## 踩坑点

- **Discord intent 忘开**：症状是消息“到了但内容为空”，agent 答非所问，日志还全绿。排查半天最后发现是个开关。
- **MarkdownV2**：直接让模型输出 MarkdownV2 会在下划线和括号上翻车，改走 HTML parse mode 或交给适配层统一转义。
- **长回复静默截断**：没做分块时，超限的后半段直接丢，用户以为 agent 说完了。
- **身份分裂**：解法是在 memory 里写一条映射说明（哪个 Discord ID 与哪个 TG 联系人是同一人），比硬编码 allowlist 柔性得多。
- **回环**：确认 bot 忽略自身与其他 bot 的消息；TG 群除了 privacy mode，还要处理 `/command` 前缀的重复响应。
- **语音条**：Discord 文字频道没有原生语音条，统一先过 Whisper 转写再进 agent，两边行为才一致。
- 本地开发用 polling 没问题，公网部署换 webhook 时记得设 secret_token，否则会有陌生人喂消息进来。

## 可复用建议

- **Agent 层保持平台无关**，一切平台差异下沉到 adapter——这是将来能不能低成本接第三个平台的分水岭。
- 路由表当配置管理，进 git，有变更记录。
- 出站加一个简单队列：Discord 每频道限速约 5 条/5 秒，重试交给队列，别让发送逻辑裸奔。
- 日志加 channel 前缀（`[tg]` / `[dc]`），排障效率差一个量级。
- 上线前用两个小号各跑一轮固定测试集：长文、代码块、图片、语音、@提及，五分钟换一个晚上安睡。

## 总结

一个 Agent 服务两个平台，代码量其实很小——channel 抽象已经把协议层吃掉了。真正的成本集中在四个策略决策：**会话边界、身份映射、格式中转、主动消息路由**。把这四件事想清楚并用配置固化下来，接第三个平台基本只是再加一份 token。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/7d1e82eb473f0ea8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/7f590d318a0cf5fd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/5d45326a4d10a4db.png)

