---
title: 一套 Agent 同时值守 Telegram 和 Discord：消息路由的工程化做法
feedId: 40264
source: 综合讨论
publishedAt: 2026-10-03
---

## 背景

我们在 OpenClaw 上跑了一个常驻 Agent，最早只接了 Telegram。后来协作群迁到 Discord，群里开始出现「这事你去那边问机器人」的尴尬。与其维护两份配置、两套记忆，不如让同一个 Agent 同时服务两个平台。这篇记录一下路由层的做法和踩过的坑。

## 问题

最容易想到的方案是起两个 bot 实例，各自调模型。实际跑起来问题不少：

- **上下文分裂**：同一个用户在 TG 问了一半，去 DC 续问，Agent 完全失忆；
- **成本与行为漂移**：两份 system prompt 迟早不一致；
- **平台差异被硬编码进业务逻辑**：长度限制（Telegram 4096 / Discord 2000）、markdown 方言、回复/提及语义各不相同。

结论很明确：路由和格式适配应该在 Agent 之外解决，Agent 内核只面对一种「规范化消息」。

## 做法

整体思路是**薄适配层 + 单一 Agent 内核**：

1. **统一会话标识**。两个 channel 适配器把消息映射成 `{platform}:{chat_id}`，再路由到同一个 agent 会话；需要跨平台记忆时，把共享记忆放到 memory store，而不是绑定某个平台 ID。
2. **入站规范化**。适配器入口统一转内部格式：sender、text、attachments、reply_to、trigger 类型（mention / reply / dm）。业务层永远不感知平台。
3. **出站格式化**。Agent 回复先输出统一 markdown，出站适配器负责：按平台长度上限切块（切在代码块边界而不是行中间）、Telegram 做 `MarkdownV2` 转义、Discord 保留 fenced code block。
4. **触发与路由规则**，配置驱动（示意）：

```yaml
channels:
  telegram:
    trigger: ["dm", "reply_to_bot", "mention"]
  discord:
    trigger: ["mention", "dm"]
routing:
  session: shared          # 两平台写入同一会话
  allowlist:
    - "telegram:-1001234..."
    - "discord:9876..."
```

5. **部署形态**。本地开发用 long polling 最省事；上服务器后换 webhook + 反向代理，Telegram 的 secret_token 和 Discord 的交互签名都要验，不要裸奔。

## 踩坑点

1. **身份合并要谨慎**。同人跨平台 ID 不同，自动合并听起来美好，误合并后两个群共享记忆会很难看。我们默认不合并，只共享项目级 memory，用户级记忆隔离。
2. **并发写会话会撞车**。TG 和 DC 的消息可能几乎同时到达，给会话加一个 per-session 串行队列就够，别一上来就上消息总线。
3. **回复循环**。接第二个 bot 后，Agent 会把别的 bot 消息当用户输入接话。所有入站消息先过滤 `is_bot`，并忽略自己的 echo。
4. **频控**。Discord 的 rate limit 按路由桶计算，Telegram 群组约 20 条/分钟。出站必须过队列 + 429 退避，切块消息尤其容易触发。
5. **MarkdownV2 转义**。`_ * [ ] ~` 等符号不转义直接 400，是 TG 出名的坑。建议默认 `parse_mode: HTML`，能省一半调试时间。

## 可复用建议

- 适配器只做「翻译」不做「思考」，该不该回、回给谁，全部收敛到 Agent 层；
- 先规范化、后格式化，中间态只有一种；
- 路由表放配置不放代码，接新平台 = 一个新适配器 + 几行路由；
- 入站消息落盘（哪怕本地 sqlite），排查「它为什么回了这句」时是救命稻草；
- 每个 channel 单独埋点：消息量、失败数、p95 延迟分开看，问题基本能定位到具体平台。

## 总结

跨平台不是「多接一个 bot」，而是把**平台差异**和 **Agent 逻辑**拆干净。适配层薄、内核单一、格式化后置——做到这三点，之后接第三个平台（Matrix、Slack）基本就是照抄一个适配器的功夫。欢迎在社区贴出你们的路由表配置，互相抄作业。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/5c60f6276ad510fc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/a48ddc72003e768c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/bcb29528d546c1a5.png)

