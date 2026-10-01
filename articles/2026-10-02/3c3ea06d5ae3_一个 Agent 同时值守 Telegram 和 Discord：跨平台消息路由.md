---
title: 一个 Agent 同时值守 Telegram 和 Discord：跨平台消息路由实战
feedId: 40051
source: 综合讨论
publishedAt: 2026-10-02
---

## 背景

用户一半在 Telegram、一半在 Discord，是很常见的处境。我们最初跑了两个独立 Agent 实例，各自挂一个平台，两周后问题就暴露了：两边记忆不同步，同一个用户在 A 平台说过的事，到 B 平台要重讲一遍；prompt 和插件配置改一次要同步两处；排障时日志分散在两个进程里。

结论很直接：合并成一个实例，让同一个 Agent 同时挂两个 channel，路由层负责隔离与分发。

## 问题拆解

合并要解决四件事：

1. **消息接入**：两个平台协议不同（Telegram Bot API 长轮询，Discord 走 Gateway WebSocket），都要进同一个消息处理链路；
2. **会话隔离**：不同平台、不同群/私聊的上下文不能互相污染；
3. **回复路由**：Agent 的输出必须回到消息来源的那个平台；
4. **格式与媒体差异**：Markdown 方言、消息长度上限、附件和语音格式各不相同。

## 做法

**1. 双 channel 接入同一个 Agent。** 在配置的 `channels` 下同时启用 `telegram` 和 `discord`，不要建第二个 agent。Telegram 侧用 BotFather 建 bot；Discord 侧在开发者后台建应用并打开 Message Content Intent。两个接入都是出站连接，不需要公网入站端口，家用网络也能跑。

**2. 会话按 channel + peer 隔离。** session key 带上 channel 和会话来源（私聊对象或群 ID），保证 Telegram 群和 Discord 频道的上下文互不干扰。需要跨平台共享的知识，显式放进 workspace 的共享 memory 文件——默认隔离，显式共享。

**3. 声明式路由规则。** 用白名单控制 Agent 响应哪些群和频道；Discord 里设 @mention 触发，Telegram 私聊直接响应、群聊走白名单。规则写在配置里，可 diff、可回滚，不要散落在代码或 prompt 里。

**4. 格式收敛到输出层。** 长度截断、转义、附件上传这类平台差异，统一放在 channel 适配层处理，prompt 只产出干净的文本，不感知平台。

## 踩坑点

- **Telegram 群隐私模式**：默认收不到普通群消息，要么在 BotFather 里关掉 privacy mode，要么把 bot 设为群管理员。这是最容易漏的一步。
- **Discord 2000 字符上限**：长回复必须分段发送，代码块跨段时注意闭合，否则渲染全坏。
- **Telegram MarkdownV2 转义**：下划线、点号、括号都可能炸格式，别手写转义，用现成库。
- **typing 状态过期**：工具调用超过十几秒，Discord 的“正在输入”会消失，长任务要做周期刷新，否则用户以为挂了。
- **session 串台**：早期偷懒共用 session，结果两边上下文互相覆盖。老老实实按 channel + peer 分。
- **旧实例残留**：迁移后忘停旧进程，同一个群收到两条回复。上线前先确认旧实例已下线。

## 可复用建议

- 平台差异全部收敛到适配层，Agent 逻辑保持平台无关——将来接 Slack 只是多加一个 channel 的事。
- 日志统一打上 `channel / peer / session` 标签，跨平台排障效率完全不同。
- 灰度顺序：先私聊 → 单群白名单 → 多群，每一步观察几天再扩。
- 身份归一（两平台 user id 映射到同一用户）先别急做，等路由稳定后再加，避免一次引入太多变量。

## 总结

跨平台不是“多接一个 API”，核心是把**接入**和**智能**分层：Agent 只关心对话与工具，平台差异交给路由和适配层。做完之后最大的收益不是省了两份配置，而是上下文终于只有一份——用户在哪问，Agent 都记得。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/16fb69da953e8301.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/839533128d14265c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/cb1c4709766d64f9.png)

