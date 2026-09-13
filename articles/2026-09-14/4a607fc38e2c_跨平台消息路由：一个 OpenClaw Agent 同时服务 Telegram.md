---
title: 跨平台消息路由：一个 OpenClaw Agent 同时服务 Telegram 和 Discord
feedId: 37443
source: 综合讨论
publishedAt: 2026-09-14
---

记录一次把单平台 Agent 扩成双平台的过程，重点不在 Agent 本身，而在路由层设计。

## 背景

社区最早只有 Telegram 群，OpenClaw Agent 在里面做答疑和日常自动化。后来 Discord 侧的用户多起来，摆在面前两条路：再起一个独立 Agent，或者让现有 Agent 同时接两个平台。前者意味着两套配置、两份记忆、两套 MCP 工具，人设和知识很快就会漂移。我们选了后者，把功夫花在消息路由层。

## 问题

拆开是三件事：

1. **协议差异**：Telegram 走 Bot API（长轮询或 Webhook），Discord 走 Gateway WebSocket；长度上限（4096 vs 2000）、Markdown 方言、附件接口全不一样。
2. **会话归属**：消息进来后，Agent 要知道回在哪个会话；同一个人在两个平台算不算同一个身份，需要明确策略。
3. **稳定性**：两边限速规则不同，断线重连和消息重复都真实踩过。

## 做法

**① 统一信封。** 入站消息先被适配器归一化成内部结构：`{platform, chat_id, user_id, msg_id, text, attachments}`。Agent 核心只认信封，不认平台。

**② 薄适配器。** Telegram 用长轮询（不需要公网回调地址，内网部署最省事）；Discord 用 Gateway，自己管心跳和断线恢复。鉴权、代理、重连逻辑全部关在适配器里。

**③ 会话键。** `session_key = platform + ":" + chat_id`，私聊场景再叠加 user_id。跨平台身份合并做成显式指令绑定，默认隔离。

**④ 出站规范化。** 回复按平台上限切片，在段落边界切而不是按字符硬切；Markdown 转成各平台安全格式；附件走各自上传接口。Agent 永远只产出"一份完整回复"。

**⑤ 限速与幂等。** 每平台一个令牌桶；用 `(platform, msg_id)` 加 TTL 缓存做去重，专门扛重连后的消息重放。

## 踩坑点

- **Discord 2000 字符上限**：超长直接发送失败。必须在段落处切片，按字符硬切会把代码块拦腰截断。
- **Telegram MarkdownV2 转义**是著名的坑，我们最终让回复默认走 HTML 或纯文本，Markdown 仅在白名单场景启用。
- **Discord 重连重放**会送回一批已处理消息，没做幂等前 Agent 会重复回答，群友看得一清二楚。
- **代理别配全局**：Telegram 需要代理，Discord 不需要，全局代理直接把它搞挂。代理设置放进各适配器内部。
- **群聊会话键踩雷**：早期只按 user_id 键控，A 群的问题答到了 B 群。会话键必须包含 chat_id。
- **同会话并发**：两个平台同时触发同一 session 导致回复乱序，给 session 加串行队列解决。

## 可复用建议

1. Agent 核心保持渠道无关，平台怪癖全压进适配器；以后接 Slack、QQ 只是再加一个薄适配器。
2. 先写一个 CLI 回环适配器在本地验证 Agent 逻辑，再接真实平台，调试噪音小得多。
3. 日志统一带 `platform:chat_id` 前缀，一条对话能从头追到尾。
4. 身份合并是产品决策不是技术默认值，默认隔离，避免 A 平台的上下文泄漏进 B 平台。
5. 凭证走环境变量，渠道独立开关，坏一个不拖垮另一个。

## 总结

这套方案的核心就两点：**归一化**与**路由**。适配器吸收平台差异，会话层决定消息归属，Agent 本体一行不改。稳定跑了一段时间，两个平台共享同一份记忆和工具链，没有出现过串台。下一步打算把信封结构固化成 OpenClaw 插件规范，踩过同样坑、手里有 Discord 限速实测数据的同学欢迎跟帖补充。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/4ed569cb3c11c7f6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/04b32d9753230806.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/e619ed49d5a44f63.png)

