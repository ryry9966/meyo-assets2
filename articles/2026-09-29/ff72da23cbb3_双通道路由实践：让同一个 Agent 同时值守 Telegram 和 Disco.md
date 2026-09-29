---
title: 双通道路由实践：让同一个 Agent 同时值守 Telegram 和 Discord
feedId: 39580
source: 综合讨论
publishedAt: 2026-09-29
---

# 背景

社区用户一半在 Telegram、一半在 Discord，是很多开源项目的常见局面。最直觉的做法是跑两个 Agent 实例、各接一个平台，但跑一阵就会发现问题：两份提示词要同步改、两份记忆各自漂移、同一个问题在两边得到不同答案。我们的目标反过来：**一个 Agent 核心，两个通道适配器**——路由层薄，核心厚。

# 问题拆解

真正要解决的只有四件事：

1. **会话键**：不同平台的消息如何映射到同一个会话存储；
2. **渲染**：核心输出的统一格式如何变成各平台的 Markdown 方言；
3. **限流与去重**：两个网关各自的速率限制和重连重放；
4. **身份**：同一个人在两个平台算一个用户还是两个。

# 做法与步骤

**第一步：统一内部消息模型。** 定义一个与平台无关的结构：`author / chat / msg_id / reply_to / text / attachments`。适配器只做协议翻译——Telegram 的 long polling、Discord 的 gateway websocket 都被压平成这个模型，业务代码感知不到平台。工具层走 MCP，两个通道天然共享同一套工具和权限配置，不用重复接入。

**第二步：分层会话键。** 路由配置（示意）用模板拼键：

```yaml
routing:
  session_key: "{platform}:{chat_id}"
  # Discord 线程场景再追加 :{thread_id}
  identity_merge: false
```

`identity_merge` 先关掉：跨平台身份合并涉及权限和隐私，不值得在第一阶段做。

**第三步：出站渲染器注册表。** 核心只产出受限 Markdown，每个平台一个渲染器，渲染失败统一降级 plain text。格式问题永远不该阻塞回复。

**第四步：入口去重 + 出站限流。** 幂等键取 `platform + msg_id`，在会话入口查重；每个通道挂独立的令牌桶队列。

**第五步：可观测性。** 日志统一带 `platform=` 标签，两条通道的延迟、错误率分开统计。排障时能一眼区分是平台问题还是核心问题。

# 踩坑点

- **Telegram 的 MarkdownV2 转义地狱**。`_ * ( ) . !` 等十几个字符漏转义就直接 400。转义放渲染器里做，且必须带 plain 降级。
- **分段切坏代码块**。Discord 单条 2000 字符、Telegram 4096。分段要按段落边界切，硬切会把代码块的 fence 切成未闭合，Discord 渲染直接乱掉。
- **重连重放导致重复回答**。Discord gateway 重连后会重放事件，Telegram 靠 `update_id` 幂等。入口去重没做的话，一次提问答两遍——而且通常是用户先发现的。
- **广播打爆限流**。往多个 Discord 频道群发时，5 条/5 秒的每频道限制很容易触发，必须排队，而不是在主流程里 sleep。
- **两份记忆漂移**。这是最初拆两个实例的老问题，解法不是合并会话，而是加一层共享的长期记忆/事实库，会话本身保持隔离。

# 可复用建议

- 适配器薄、核心厚：平台差异全部关在适配器和渲染器里；
- 内部消息模型先行，先定结构再写第一个适配器；
- 每个通道独立的 feature flag 和限流器，出问题可以单独关一边；
- 写一个输出到 stdout 的 null adapter，回归测试不依赖真实 token；
- 身份合并晚点做，先用共享记忆过渡，成本最低。

# 总结

跨平台路由没有黑魔法，核心是控制面和数据面的分离：路由层要笨，只做映射和翻译；核心要稳，对平台无感知。把转义、分段、去重这些脏活分层放好之后，再加第三个平台（Slack、Matrix 之类）基本就是多写一个适配器的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/59b4c180ff741f44.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/1dfa38e691ca224c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/1e7e2673e16244ad.png)

