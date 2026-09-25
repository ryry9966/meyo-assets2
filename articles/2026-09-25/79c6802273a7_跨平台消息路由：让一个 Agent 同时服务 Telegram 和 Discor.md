---
title: 跨平台消息路由：让一个 Agent 同时服务 Telegram 和 Discord
feedId: 38929
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

最早只有一条 Telegram 群的 bot，后来协作迁到 Discord，bot 需求跟着过来。第一反应是复制一套，但两份 prompt、两份工具配置、两处记忆，很快出现"两边答案不一致"的问题。OpenClaw 的设计里 channel 只是适配层，agent core 只处理统一的内部消息格式，所以正确做法从来不是跑两个 agent，而是一个 agent 挂两条 channel。

## 问题

真正的工作量不在"接 SDK"，而在抹平平台差异：

- **会话粒度**：Telegram 按 chat 隔离即可，Discord 多一层 thread，thread 要不要继承频道上下文需要显式决策；
- **输出方言**：Telegram MarkdownV2 的转义规则和 Discord markdown 几乎是两套语言，长度上限也是 4096 对 2000；
- **速率限制不对称**：Telegram 单 chat 约 1 msg/s，Discord 每频道每 5 秒约 5 条；
- **身份不通**：两边的 user id 是两套体系，权限和记忆绑定要做映射。

## 做法

1. 两个 channel 插件指向同一个 agent 实例，prompt、工具、MCP 配置只维护一份：

```yaml
agents:
  main: { workspace: ./agents/main }
channels:
  telegram:
    bot_token: ${TG_TOKEN}
    agent: main
    allowed_chats: ["-100xxx"]
  discord:
    bot_token: ${DC_TOKEN}
    agent: main
    mention_only: true
routing:
  session_key: "{platform}:{chat_id}"
```

2. 会话键用 `{platform}:{chat_id}`，**默认不跨平台共享记忆**。需要共享时走显式白名单，避免无意间把 A 平台的上下文泄到 B 平台。

3. 所有平台怪癖下沉到出口适配层：格式转换、按上限分段、媒体上传，agent 永远只输出标准 markdown。

4. 群聊触发策略分开配：Discord 侧开 `mention_only`，Telegram 侧用命令前缀 + @提及，否则高频群会把 token 烧穿。

## 踩坑点

- MarkdownV2 转义是无底洞。我们的结论是放弃它，Telegram 侧降级到 HTML parse mode，由 adapter 做一次转换，别指望 LLM 输出转义正确的文本；
- 长回复分段会把代码块拦腰切断。分段前先按代码围栏切分，代码块整体保留在单段内，超长就直接发文件；
- Discord 的 interaction token 15 分钟过期，长任务别 defer 干等，先落普通消息再编辑；
- 轮询/回调重试会造成重复回复，入站消息要有去重键（platform + message id），在 adapter 层拦截；
- 出站要有队列 + 合并：agent 连续输出多条时合并成一条再发，同时兜住两边限频。

## 可复用建议

- 判断架构是否健康的标准：将来接第三条 channel 时，agent 目录是否一行都不用改；
- 先用 echo agent 打通全链路（入站→路由→出站→分段→限频），再接真模型，能省大量排查时间；
- 路由决策打结构化日志：哪条 channel、命中哪条规则、映射到哪个 session，排障时全是线索；
- 身份映射表单独存，权限挂在映射层而不是 agent 层。

## 总结

跨平台路由的本质是把"平台差异"和"Agent 逻辑"解耦。OpenClaw 的 channel 适配层天然适合承接这些脏活：会话键、格式转换、限频、去重都收在 adapter 和 gateway 里，agent 保持一个纯粹的大脑。做完之后最直观的收益不是省了一个 bot 进程，而是两边用户面对的是同一个"它"，答案和行为终于一致了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/20dc3120452ff0e4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/d8722d7fae52fc46.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/4016e324617bde50.png)

