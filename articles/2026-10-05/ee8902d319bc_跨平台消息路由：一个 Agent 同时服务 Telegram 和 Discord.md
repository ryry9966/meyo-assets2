---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 40544
source: 综合讨论
publishedAt: 2026-10-05
---

## 背景

我们的 Agent 最初只挂在 Telegram 上，服务自己和一个十来人的小组。后来协作搬到了 Discord，第一个反应是再部署一份、两边各管各的。跑了一周就发现维护两份配置、两份记忆、两套提示词的成本太高，于是回头研究一条路：同一个 Agent 核心同时绑定两个 channel。

OpenClaw 的 gateway 本来就把 channel 和 agent 解耦了，这件事架构上是支持的。真正的工作量不在"接上"，而在路由和格式这两层。

## 问题

接两个 bot 只是一半工作，实际跑起来会遇到四类问题：

1. **会话隔离**：Telegram 私聊和 Discord 频道如果共享同一个 session，两边上下文会互相污染。
2. **格式差异**：Telegram 的 MarkdownV2 转义苛刻；Discord 是另一套 markdown 方言，代码块、引用、长文折叠行为都不同。
3. **消息限制**：Telegram 单条 4096 字符，Discord 2000，超长回复必须有切分策略。
4. **触发规则**：私聊默认全响应，群聊要靠 @ 或前缀，两个平台的群语义（Discord 的 thread、Telegram 的回复）也不对齐。

## 做法

我们的分层原则：channel 适配层只做收发和格式转换，路由与业务判断全部收敛在 agent 侧。

1. 在 gateway 配置里同时声明 telegram 和 discord 两个 channel，各自填 token，绑定同一个 agent。
2. session key 用 `channel:chat_id` 粒度，如 `telegram:12345` 与 `discord:98765` 天然隔离；跨平台想共享知识走长期记忆，而不是共享 session。
3. 在系统提示里注入"当前消息来自哪个 channel"，让模型知道回复语境（Discord 可以引导到 thread，Telegram 群里尽量短）。
4. 超长回复在发送层按目标平台限制切分，切分点优先选代码块边界和段落。
5. 触发规则全局统一：@提及或固定前缀，两个平台行为一致，降低使用心智。

配置思路示意（字段名以你本版本文档为准）：

```json
{
  "channels": {
    "telegram": { "botToken": "..." },
    "discord": { "token": "...", "intents": ["guilds", "message_content"] }
  },
  "agents": { "default": { "workspace": "~/agent" } }
}
```

## 踩坑点

- **Discord 不开 Message Content Intent**，bot 在群里只能收到 @ 它的消息，表现为"时灵时不灵"，先查这个。
- **Telegram 群隐私模式**：bot 默认收不到普通发言，要么关掉隐私模式并把 bot 重新拉进群，要么接受只有命令和 @ 触发。
- **MarkdownV2 转义是重灾区**，下划线、点号、括号都会炸。我们最终 Telegram 侧降级为纯文本加代码块，Discord 侧保留完整 markdown。
- **切分不要放进模型**。让模型输出完整内容，发送层负责切，否则模型会为了凑长度牺牲回答质量。
- **typing indicator 别两边都发**，长任务时显得很吵，我们只在任务超过 3 秒时发一次。

## 可复用建议

- 把平台差异当成适配层的实现细节，agent 侧只处理统一的消息信封：来源、会话、文本、媒体、回复目标。
- 日志按 `channel + chat_id` 打标，排障时能还原一条消息从收到答的完整链路。
- 灰度上线：先只开一个测试群，验证触发规则和格式，再放开私聊。
- 命令集两边保持一致，文档写"这个 Agent 怎么用"，而不是分平台写两份。

## 总结

一个 Agent 服务多平台，架构上不难，难的是把平台差异收敛到适配层、把路由策略想清楚。我们的三条核心经验：session 按 channel+chat 隔离、格式转换下沉到发送层、触发规则全局统一。做到这三点，后续加第三个平台基本只是新增一段 channel 配置的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/66260a50f47c3703.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/44c875b28cb9c70a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/11006590fd5e1a4d.png)

