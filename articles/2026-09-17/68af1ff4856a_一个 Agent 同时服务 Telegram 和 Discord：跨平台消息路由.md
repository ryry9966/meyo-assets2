---
title: 一个 Agent 同时服务 Telegram 和 Discord：跨平台消息路由的工程实践
feedId: 37992
source: 综合讨论
publishedAt: 2026-09-17
---

## 背景

Agent 跑起来之后，用户不会迁就你的入口。我们团队一部分人泡在 Telegram，另一部分在 Discord。如果两边各跑一个 Agent 实例，很快会出现上下文分裂、配置漂移、重复消耗 token 的问题。收敛成"一个 Agent + 两个通道适配器"之后，这些麻烦基本消失，代价是路由层要自己写。

## 问题

拆开看，跨平台路由要解决四件事：

1. **消息归一化**：Telegram 的 update 和 Discord 的 gateway event 结构完全不同，必须先转成统一的内部消息信封；
2. **会话身份**：同一个真人可能两边都用，是合并成一个 session 还是分开？我们选择先分开（`platform:user_id` 作为 session key），避免误合并；
3. **回复回程**：Agent 的回答必须原路返回，不能串台；
4. **能力差异**：Discord 的 interaction token 有 3 秒超时，Telegram 分 webhook 和 polling 两种模式，Markdown 方言也不同。

## 做法

核心是一个轻量路由层，分三步搭：

**第一步：定义消息信封。** 所有入站消息统一为 `{platform, chat_id, user_id, session_key, content, attachments, reply_to}`。适配器只做协议解析和封装，不含任何业务逻辑。

**第二步：适配器实现统一接口。** Telegram 侧用 long polling（自托管场景比 webhook 省事，不用管证书和公网入口）；Discord 侧走 gateway websocket。各自处理 ACK：Discord 收到指令先 `defer` 应答再慢慢生成，Telegram 对 polling 直接返回 200 即可。

**第三步：路由与回程。** 路由层按 `session_key` 取 Agent 会话，跑完后从信封读 `reply_to` 决定回哪个通道。回复渲染按平台分发：Telegram 用 HTML 子集，Discord 用自己的 Markdown，代码块两边都保留围栏格式。

权限映射单独维护一张表：`platform + chat_id -> 等级`，Agent 调 MCP 工具前先查表，避免 Discord 某个公开频道把高权限工具暴露出去。

## 踩坑点

- **Discord 3 秒超时**：LLM 生成经常超过 3 秒，必须先 defer 再 edit，否则前端直接报"应用无响应"。这是踩得最狠的一个。
- **Markdown 方言**：Telegram 不支持 `##` 标题，Discord 的链接写法又不一样。别指望一套渲染通吃，老老实实做按平台的 formatter。
- **长消息分段**：Telegram 单条 4096 字符，Discord 2000。超长回复要切分，且不能把代码块劈成两半——按围栏边界切。
- **session 串扰**：最初消息队列是全局单消费者，A 平台高峰会阻塞 B，改成按 `session_key` 分区后才恢复。
- **附件**：两边大小限制和下载方式不同，统一走"先落对象存储、信封只带 URL"的方案，省掉大量分支判断。

## 可复用建议

- 适配器保持"薄"，协议解析和业务彻底分离，之后接 Slack、飞书只是再写一个 adapter；
- session key 永远带上 platform，除非有明确的身份打通机制，否则不要跨平台合并会话；
- 各平台的 rate limit 在适配器内部消化，别让路由层感知平台差异；
- 路由层保持无状态，会话状态交给 Agent 自己的 store，这样路由层随时可以水平扩。

## 总结

跨平台路由的本质不是"多接一个 SDK"，而是把平台差异全部封进适配器，让 Agent 面对一份归一化的消息流。先把消息信封和 session key 这两个契约定稳，后面的通道扩展都只是体力活。这套结构在我们这边稳定跑了两个月，新增一个通道约一天工作量，维护成本完全可控。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/e12f572dc5d25d66.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/8acbc48d358b781f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/94e435deca7909a5.png)

