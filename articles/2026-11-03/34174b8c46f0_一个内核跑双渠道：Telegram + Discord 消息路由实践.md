---
title: 一个内核跑双渠道：Telegram + Discord 消息路由实践
feedId: 43309
source: 综合讨论
publishedAt: 2026-11-03
---

## 背景

我们的 Agent 最初只接了 Telegram，用于内部值班答疑。后来 Discord 侧的社区成员也想要同样的能力。第一反应是再部署一个实例，两周后就出现了配置漂移：提示词改了 Telegram 侧忘了 Discord，记忆和技能配置完全分裂。这篇帖记录我们把两条渠道收敛到同一个 Agent 内核下的做法，思路对其他渠道（Slack、飞书等）同样适用。

## 问题拆解

跨平台表面上是"多接一个 SDK"，实际是三类问题：

1. **协议差异**：Markdown 方言、长度上限（Telegram 4096 / Discord 2000）、线程与话题语义、附件大小限制都不一样；
2. **会话归属**：同一个人在两个平台算不算同一个会话，键怎么定；
3. **运维细节**：webhook 还是轮询、限流、重复实例冲突。

## 做法

**第一步，先定统一信封，再写适配器。** 所有入站消息不管来自哪个渠道，都规整成同一结构：`platform`、`chat_id`、`user_id`、`text`、`attachments`、`reply_to/thread_id`、`message_id`。渠道插件只做翻译，不做业务判断。

**第二步，路由层按 `platform:chat_id` 生成会话键，投递给同一个 Agent 内核。** 内核完全不感知渠道存在。提示词、工具（含 MCP 工具）、技能只有一份，改一处两边生效。

**第三步，出站反向适配。** 回复先过格式器：按平台转 Markdown 方言（Telegram 我们直接用 HTML parse mode，避开转义地狱）、超长切片，且切片位置避开代码块中部。

**第四步，身份合并放后期。** 先按 `platform:chat_id` 隔离会话跑稳，确有需求再加一条绑定命令，人工验证后把两个键映射到同一 profile。不要第一天就做自动合并，错合比不合难收拾。

## 踩坑点

- 同一个 Telegram token 被两个实例同时 long polling 会触发 409 冲突，表现为消息随机丢失。升级或调试前务必先停旧进程。
- Discord 的 2000 字符限制切出的多段消息，发送顺序可能乱：分段要带序号，出站队列串行发送。
- Telegram MarkdownV2 转义规则极繁琐，漏一个字符整条消息被拒，用 HTML 模式一劳永逸。
- `reply_to`/`thread_id` 在部分转发路径上会丢，导致 Discord 回复落错线程。信封关键字段要做校验，别设成可选。
- 附件：Discord 10MB、Telegram bot API 20MB，超限在适配器直接拒绝并提示，不要把大二进制塞进内核上下文。
- 限流：Discord 单频道约 5 条/5 秒，Telegram 单聊约 1 条/秒。出站必须有队列加退避，重试用 `message_id` 做幂等去重。

## 可复用建议

- 信封 schema 先行，适配器保持"笨"，平台怪癖全部留在适配层，不要泄漏进提示词。
- 写一个 stdout 空渠道适配器用于本地联调，不碰真实 token。
- 每条消息记录 platform、耗时、token 消耗；没有分渠道指标，就不知道该优化哪边。
- 先 polling 后 webhook。webhook 需要公网 HTTPS，家庭环境自托管时轮询反而省事。
- 两个渠道分开配 allowlist 和凭据，一个 token 泄露不要殃及另一边。

## 总结

跨平台路由的本质是把"消息进出"与"Agent 决策"解耦：内核一份，适配器若干，会话键和限流出站队列是两块稳定基石。身份合并这类锦上添花的功能，等单平台稳定跑两周再上不迟。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/c1dde94956018cec.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/4023236f5c4b8e0c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/ac9c14e3e8af076e.png)

