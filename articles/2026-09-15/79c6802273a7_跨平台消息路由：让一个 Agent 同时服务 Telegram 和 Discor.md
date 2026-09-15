---
title: 跨平台消息路由：让一个 Agent 同时服务 Telegram 和 Discord
feedId: 37681
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

我们的 Agent 最初只挂在 Telegram 上，做内部运维问答。后来社区迁到 Discord，问题来了：同一个人维护两套 Agent 实例，prompt 改一处忘一处，知识库更新不同步，排查问题时日志分散在两个进程里。目标很朴素：**一个大脑，两个入口**——Agent 核心只有一份，Telegram 和 Discord 都只是它的“前端”。

## 问题

两个平台的差异比想象中大，集中在三层：

1. **协议层**：Telegram 用长轮询或 Webhook，Discord 走 Gateway WebSocket，事件模型完全不同；
2. **身份层**：`chat_id / user_id` 体系和 Discord 的 `guild / channel / user` 对不上，会话怎么绑定是个设计决策；
3. **表达层**：Markdown 方言不一致、消息长度上限不同（Telegram 4096，Discord 2000）、@提及格式各异。

如果在 Agent 里直接写 `if platform == "discord"`，两周后就会失控。

## 做法

核心思路是**信封 + 适配器 + 渲染器**，把平台差异全部挡在 Agent 外面。

**第一步：定义统一消息信封。**

```json
{
  "platform": "telegram | discord",
  "route_key": "telegram:chat_12345",
  "user_id": "u_678",
  "msg_id": "m_009",
  "text": "...",
  "attachments": []
}
```

**第二步：两个适配器只做收发。** Telegram 适配器走 Webhook，Discord 适配器连 Gateway，收到事件后统一转成信封入队。适配器不做任何业务判断。

**第三步：路由层按 `route_key` 绑定会话。** 同一个 `platform:chat_id` 始终哈希到同一个 Agent 会话，保证上下文连续。注意：**跨平台默认不合并身份**——同一个人在两边就是两个会话，合并身份涉及权限和隐私，不值得第一版就做。

**第四步：出站渲染层。** Agent 输出统一的内部 Markdown，渲染器按平台转换：Telegram 转 HTML 并转义特殊字符；超长消息在渲染层统一分片，Agent 完全不感知长度限制。

**第五步：可靠性兜底。** 入站消息以 `platform + msg_id` 作幂等键；出站对各平台的 429 分别做指数退避。

## 踩坑点

- **重复回复**：Webhook 重试叠加 Discord 断线重连补发事件，一晚上回了几十条重复消息。幂等键必须最先做，不是最后。
- **MarkdownV2 转义**：Telegram 的 `_ * [ ]` 不转义直接 400 报错。建议第一版只支持一个最小子集，别贪。
- **@提及**：Discord 用 `<@id>`，Telegram 用 mention 实体，两个平台都只能在渲染层各自生成，Agent 输出里用占位符。
- **编辑事件**：Discord 有 message update，Telegram 也有 edited_message。第一版直接忽略编辑，只处理新建，功能砍掉反而稳定。
- **限速**：两平台的 429 各自独立退避，别共享一个全局限速器，否则一边慢一边陪葬。

## 可复用建议

- 适配器接口收敛到三个能力：`on_message / send / split_message`，后来接第三个平台只花了半天；
- `幂等键` 和 `route_key` 是这套架构的两根支柱，换平台也通用；
- 监控盯三个指标：每平台收发计数、渲染转换失败数、队列延迟；
- 新平台灰度时先做“只读”（只收不发或只回显），观察一周再放开完整回复。

## 总结

跨平台不是“多接一个 Bot”的问题，而是把平台差异从 Agent 逻辑中剥离出来的架构问题。信封统一数据、适配器隔离协议、渲染器隔离表达，Agent 核心保持平台无关——之后无论加 Slack 还是 Matrix，都是套模板的体力活，而不是重构。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/48a1aa2ad9c44d24.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/b8f0a55fa7440276.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/508ac2d1f1bcc826.png)

