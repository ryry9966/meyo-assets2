---
title: 一个 Agent，两张嘴：Telegram + Discord 双通道消息路由实践
feedId: 38270
source: 综合讨论
publishedAt: 2026-09-20
---

## 背景

OpenClaw 的架构里，Channel（通道）和 Agent 本来就是解耦的：Agent 只负责思考，Channel 负责收发。我的场景是个人知识库问答 Agent——朋友群在 Telegram，项目协作在 Discord，两边提问高度重叠。最初跑了两套实例，很快出现记忆不同步、配置漂移，于是合并为单实例双通道。本文记录这次合并的做法和坑。

## 问题

合并要解决三件事：

1. **路由**：入站消息如何汇到同一个 Agent，出站回复如何回到正确的平台和会话；
2. **会话与身份**：同一个人在两个平台的 session 要不要合并；
3. **格式**：Markdown 方言、消息长度上限、线程/回复模型都不一样。

## 做法

**1. 通道接入。** gateway 同一实例下同时启用 telegram 和 discord 两个 channel，各自配 token；allowlist 两边独立维护，避免一边泄漏波及另一边。Telegram 用 long polling（家用宽带没有公网 TLS，省事），Discord 走 gateway WebSocket。

**2. 路由与会话键。** 以 channel + chat id 作为 session key，各自维护独立会话；入站消息打上 platform 标签注入上下文。Agent 提示词可以感知"当前在哪个平台"，但平台特有逻辑不写进提示词。

**3. 出站归一化。** Agent 统一输出纯 Markdown，由各 channel adapter 翻译：Discord 走普通消息/embed，Telegram 转 HTML 并处理转义。分片按平台上限裁剪（Discord 2000 字符/条，Telegram 4096/条），分片逻辑必须禁止切断代码块。

**4. 身份合并（可选，后置）。** 先让每个平台独立 session 跑稳，再通过 pairing 流程把同一人的两个 session 指向同一 memory 命名空间——验证动作做成命令（两边各发一次确认），不靠用户名猜测。

## 踩坑点

- **Telegram MarkdownV2 转义是地狱。** 别让 Agent 输出 MarkdownV2，统一输出普通 Markdown，转义全部交给 adapter。下划线、点号、括号都会咬人。
- **429 处理要分通道。** Discord 限速按 bucket，Telegram 按广播频率，共享一个发送队列会被慢的一边拖垮。按通道各设队列和退避策略。
- **重试导致重复发送。** 出站消息加本地消息 id，发送前查重；Discord gateway 断线重连后容易重放事件，尤其要注意。
- **群里太吵。** 两边都配置成仅 @提及或命令触发，否则 Agent 会在闲聊里频繁插话，很快被人禁言。
- **打字状态别漏。** Discord 的 typing 指示、Telegram 的 chat action，长任务不给反馈，用户会重复发消息，触发更多任务，恶性循环。

## 可复用建议

- **Channel 当薄适配器**：转义、分片、限速、线程等平台差异全部收在 adapter 层，Agent 提示词保持平台无关，最多注入一个 platform 变量。
- **会话合并放到最后**：先用独立 session 验证路由稳定，身份合并是锦上添花，不是前置条件。
- **出站加格式 dry-run**：发送前在本地把 Agent 输出分别渲染成两平台的最终形态，转义错误在日志里就能看到，不用等用户报"消息是乱码"。
- **日志统一带 channel 前缀**：排障时一眼区分问题出在哪条链路。

## 总结

双通道接入本身不难，适配器半天就能跑通；真正的工作量在会话键设计、格式归一化、限速与重试这些管道工程上。原则一句话：**Agent 只管想，Channel 只管说，路由层管好"谁说的、说到哪、怎么说得体面"。** 后续如果再加 Slack 或 Matrix，按这套分层基本只是复制一个 adapter 的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/763ebb6bb5e953e5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/fa92573cf4427388.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/95bfd0a6c9c5515a.png)

