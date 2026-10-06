---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 40734
source: 综合讨论
publishedAt: 2026-10-07
---

## 背景

我之前的用法是分裂的：Agent 挂在 Telegram 上当私人助手，团队协作放 Discord。结果是两套部署、两份密钥、两份会话记录，同一个人在两边是两个身份，上下文完全不通用。OpenClaw 的 gateway 本身支持多 channel 并存，于是把两条链路并到了同一个 agent 实例上。结论先说：可行，比想象中省事，但"连上"和"用好"之间隔着三层工程问题。

## 三个真实问题

1. **格式与长度不对称**：Telegram 单条 4096 字符，MarkdownV2 转义极苛刻；Discord 单条 2000 字符，markdown 是另一套方言。同一份回复直接广播，必有一边崩。
2. **会话路由**：哪个 Discord 频道、哪个 Telegram chat 绑定哪个 session？没有显式规则，消息就会串台。
3. **传输行为差异**：Telegram 走长轮询，Discord 是 websocket 长连接，限流、断线、重连逻辑完全不同。

## 做法

**1. 单 gateway，双 channel。** 只跑一个 gateway 进程，两边各配 token。注意：绝不要在第二台机器上重复挂同一个 Telegram token，两个进程抢 getUpdates 会互相 409，表现为"时通时断"，很难第一眼定位。

**2. 声明式会话绑定。** allowlist 限定来源；Discord 按频道粒度绑 session，DM 单独一条；首次接入走 pairing 握手。路由规则全部放配置里（示意，字段以你所用版本为准）：

```yaml
routing:
  - match: { channel: discord, ref: "<channel_id>" }
    session: team-dev
  - match: { channel: telegram, ref: "<chat_id>" }
    session: personal
```

**3. 出站分层格式化。** agent 只产出统一的内部 markdown，每个 channel 一个 formatter 负责落地：Discord 超 2000 拆条，且先按代码块边界拆、再按行拆；Telegram 格式化失败就整体降级纯文本——宁可丢样式，不能丢回复。

**4. 媒体不对称处理。** Telegram 语音先转写再进上下文；Discord 附件走 URL 引用，超限落本地缓存。不要假设两边附件能力对称。

**5. 按 channel 打观测标签。** 首响应延迟、拆条次数、格式化降级次数、错误计数。"是平台的问题还是 agent 的问题"，靠这个区分。

## 踩坑点

- 拆条把代码块切成两半，Discord 渲染直接崩。拆分逻辑必须块级优先。
- Telegram MarkdownV2 是重灾区，未转义的下划线、点号都会让整条消息被拒。要么走 HTML parse mode，要么默认纯文本。
- 同一人两平台两个身份，记忆不共享。真要合并，在 session 层做 identity mapping，别靠往 system prompt 里塞"如果他提到 X 就认为……"这类软规则，迟早出错。
- Discord websocket 断线重连窗口内的消息会漏，靠 resume 加落库补拉兜底，别只信内存队列。
- Discord 每频道约 5 条/5 秒的限流，通知类群发必须排队，否则触发退避。

## 可复用建议

- 路由规则声明式，代码里不写死任何 channel id。
- formatter 与 transport 解耦，未来接新平台只写一个 adapter。
- 上线前先用 echo 模式把两条链路跑通，再放开真实 agent。
- 每个渠道独立告警阈值，Telegram 报警不代表 Discord 有事。

## 总结

跨平台接入的难点从来不是"连上"，而是格式、会话、限流这三层的工程化。OpenClaw 的 channel 抽象足够薄，配合声明式路由和分层格式化，一个 agent 同时服务两个平台是稳定且可维护的。建议从低风险频道灰度起，观察一周指标再全量。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/6a14f1e595d7f9d1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/61b41a9fed4573be.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/99dd90938e6839d5.png)

