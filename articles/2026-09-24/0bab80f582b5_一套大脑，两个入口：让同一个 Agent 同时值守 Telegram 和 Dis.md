---
title: 一套大脑，两个入口：让同一个 Agent 同时值守 Telegram 和 Discord
feedId: 38822
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

社区用户一部分在 Telegram，一部分在 Discord。之前我分别跑了两个 bot，各自接独立的 Agent 实例，结果很快出现三类问题：同一份系统提示词要改两处；两个平台的会话记忆完全割裂；排障时日志分散，定位问题得来回切终端。这篇记录我把它们收敛为「一个 Agent 核心 + 两个通道适配器」的过程，在 OpenClaw 下大约一个下午可以搭完。

## 问题

跨平台不是“多接一个 SDK”那么简单，难点全在中间层：

- **格式不一致**：Markdown 方言、长度上限（Telegram 4096，Discord 2000）、代码块渲染行为都不同；
- **会话归属**：按用户 ID 建 session 会在群里串味，按群 ID 建会让两个人共享一份上下文；
- **频控差异**：Discord 按通道限速，Telegram 单群约每秒 1 条，突发回复直接 429。

## 做法

架构上只有一句话：通道只负责收发，理解全部交给 Agent 核心。

1. **统一信封**。两个适配器把平台消息归一化成同一个 envelope：`platform / channel_id / user_id / text / attachments / reply_to`。Agent 核心只认 envelope，不知道消息来自哪里。
2. **Session 键设计**。`session_key = hash(platform + channel_id + user_id)`。私聊场景可省略 channel_id；如果要做跨平台身份打通，再在本地存一张 Telegram uid ↔ Discord uid 的映射表。映射是可选的，没有映射就当两个独立用户。
3. **输出渲染器**。渲染器写成纯函数：输入 Agent 产出，输出平台消息数组。长消息先按代码块边界切，再按平台上限二次切分；Telegram 侧用 MarkdownV2，转义规则集中在一处处理。
4. **发送队列**。每个通道一个带速率控制的队列，429 时按 Retry-After 退避。Agent 永远不直接发消息。
5. **OpenClaw 配置**。Agent 只定义一份，channel 插件挂两个，路由层就是上面这几十行代码，没有引入额外服务。

## 踩坑点

- **Markdown 方言是最大的坑**。Discord 吃得下的语法 Telegram 会原样吐出来；反过来 MarkdownV2 的转义能吃掉一下午。结论：Agent 输出一律用最朴素的 Markdown，方言转换全部下沉到渲染器。
- **切分消息切断代码块**。按固定长度硬切会得到两截渲染残废的代码，务必先按块切再按长度切。
- **不要依赖“编辑消息”表达状态**。两个平台的 edit 行为和通知触发逻辑不一致，状态变化宁可发新消息。
- **日志要打 envelope**。只打 Agent 内部日志，跨平台问题基本没法排——你分不清是适配器没收到，还是渲染器发歪了。
- **错误信息脱敏后再进频道**，堆栈只进日志。

## 可复用建议

- envelope 字段定下后尽量不动，加字段向后兼容，这是整个方案的生命线；
- 渲染器保持无副作用，方便用单元测试覆盖方言差异；
- 频控参数按平台配置化，别写死在队列里；
- 身份映射只存本地、只做单向提示（“你之前在另一边问过……”），不要默认合并记忆，涉及隐私；
- 每个通道做独立健康检查，单通道故障只降级，不重启核心。

## 总结

方案的本质是把“多平台”从 Agent 的职责里剥离：Agent 面对一个稳定的信封协议，平台差异全部由适配器和渲染器消化。跑了两周，两个平台日均消息量百级，队列基本零积压，改一次提示词两边同步生效。如果你也在 OpenClaw 里同时维护多平台的 bot，建议先从统一信封这一步做起，后面的复杂度会小很多。欢迎在评论区交换各自的 envelope 字段设计。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/ce9e86225d828342.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/1a86ef291fe372cc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/669c7e932ab908a1.png)

