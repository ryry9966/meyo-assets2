---
title: 一套 Agent 核心，两条链路：Telegram 与 Discord 的跨平台消息路由实践
feedId: 40615
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

我们的 Agent 最早只挂在 Telegram 上服务内部群，后来社区在 Discord 建了频道，希望同一个 Agent 也能在那边答疑。第一反应是再起一个实例，但很快发现：两个实例意味着两份记忆、两份配置、两套插件状态，改一个 prompt 都要同步两处。更合理的路线是：**一套 Agent 核心，两条接入链路**。本文记录我们用 OpenClaw 多 channel 配置加自建适配层完成这件事的过程。

## 问题

跨平台路由的难点不在“能收到消息”，而在三个不一致：

1. **消息模型不一致**：mention 格式（`<@id>` vs `@username`）、回复引用、话题/线程结构、富文本能力全都不同；
2. **会话与身份不一致**：同一个人在两个平台是两个 user id，记忆要不要打通、打通到什么程度需要设计；
3. **发送侧约束不一致**：Telegram 单条 4096 字符、每 chat 约 1 msg/s；Discord 2000 字符、按路由限速，超限行为也不同。

## 做法

**第一步，定义统一的内部消息模型。** 入站适配器把两边消息归一化成同一个结构：`platform`、`chat_id`、`user_id`、`text`、`attachments`、`reply_to`、`thread_id`。适配器只做翻译，不做业务判断。

**第二步，双 channel 指向同一 workspace。** 在配置里声明两个 channel，各持 token，共享同一个 agent 工作区和插件集。session key 由 `agent + 会话标识` 生成；另维护一份**显式的身份映射文件**，把确认是同一人的两平台账号合并到统一 user id——手工维护，默认不自动合并。

**第三步，出站渲染分层。** 模型输出统一的“结构化块”（段落、代码块、列表、附件），再由各平台 renderer 转成平台格式。Discord 直接用原生 markdown；Telegram 我们放弃了 MarkdownV2，统一走 HTML parse mode，转义问题基本消失。

**第四步，限速与排队。** 每个平台一个出站队列 + 令牌桶，按 chat 维度排队。长耗时工具调用期间主动发 typing 状态（Discord 的 typing、Telegram 的 sendChatAction），否则用户会觉得 Agent 挂了。

**第五步，媒体统一落盘。** 两边收到的附件先存本地存储，把 URL 交给模型；出站时再由适配器各自上传，绕开附件上限和 file_id 生命周期的差异。

**第六步，幂等与防回环。** 用 `(platform, message_id)` 做幂等键去重；出站前过滤目标为自身 bot id 的消息，避免中继群里自己触发自己。

## 踩坑点

- MarkdownV2 的转义是个黑洞，允许字符集极小，模型输出几乎必然踩雷，直接换 HTML；
- Discord 的 forum 频道要显式建线程，Telegram 群的 topic 必须透传 `message_thread_id`，漏掉任何一个都会把回复刷到主频道/主群；
- Discord 上编辑消息做流式更新体验很好，但 Telegram 的 `editMessageText` 有频率限制，流式只能降级成分段发送；
- 身份合并要克制：合并后跨平台上下文互见，涉及隐私，默认关闭、按需开启；
- 两边附件上限不同（大致在 10–50MB 区间），超限路径要单独处理。

## 可复用建议

1. **适配器保持“傻”，核心保持“聪明”**，平台差异全部挡在 renderer；
2. 内部 schema 加版本号，后续加平台不用改核心；
3. 做 dry-run 模式：出站只打日志不发送，排障时极其有用；
4. 能力用特性开关按 channel 控制，比如语音输入只在 Discord 开；
5. 日志里永远带 platform 标签，跨平台问题基本靠它定位。

## 总结

跨平台路由的本质不是“多接一个 bot”，而是把消息模型、身份模型、渲染策略拆成三层，让平台差异收敛在边缘。结构定好之后，加第三个平台只花了一个下午。如果你也在维护多平台 Agent，建议先花时间设计内部 schema，这比急着接平台划算得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/5d13d0e4ad64c21c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/fd769ca92fe67c6d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/2d38e6b503ff2fe5.png)

