---
title: 一个 Agent 吃两个渠道：让 OpenClaw 同时服务 Telegram 和 Discord
feedId: 39329
source: 综合讨论
publishedAt: 2026-09-28
---

## 背景

我的使用场景很典型：个人事务的日常问答在 Telegram，兴趣社区的技术讨论在 Discord。一开始是两个独立 bot，各自一套提示词、各自一份记忆，很快出现"问过的问题换个渠道再问一遍"的尴尬。目标是让同一个 OpenClaw Agent 同时挂两个渠道，共享工作区、记忆和 MCP 工具。

## 问题

多渠道接入不是多填一个 token 的事。真正要处理的是四件事：会话怎么隔离、消息格式怎么适配、两个平台的长度上限和线程语义不同、同一个人在两个平台的身份怎么对应。协议差异 OpenClaw 的渠道层会消化掉一部分，剩下的属于设计决策，得自己做。

## 做法

1. **准备凭证**。BotFather 建 Telegram bot 拿 token；Discord Developer Portal 建应用，记得打开 Message Content Intent，否则收不到正文。
2. **双渠道配置**。在 `openclaw.json` 的 channels 段同时写 telegram 和 discord 两块，allowlist 收紧：Telegram 用 allowFrom 限定用户 ID，Discord 限定 guild 和频道，避免上线即公网可用。
3. **会话策略**。默认 session key 按"渠道 + 会话"隔离，形如 `agent:main:telegram:<chatId>`。我保留了隔离，只把需要跨渠道的长期记忆放进共享工作区（MEMORY.md、笔记文件），让 Agent 自己读写。强行合并会话，两边的上下文反而会互相污染。
4. **格式适配**。约束 Agent 输出朴素 Markdown，交给渠道适配层转换；Telegram 端避开重度 MarkdownV2 语法。
5. **部署验证**。`openclaw gateway` 挂成常驻进程（systemd 托管），`openclaw doctor` 做自检，`openclaw logs` 带渠道标签观察双端收发。

## 踩坑点

- **MarkdownV2 转义**：下划线、方括号没转义，Telegram 会整条拒收或渲染崩坏。解法是 plain text 优先，或让适配层改走 HTML parse mode。
- **长度上限**：Telegram 4096、Discord 2000，适配层会自动分块，但代码块被拦腰截断很难看。长回复让模型主动分段，别依赖分块兜底。
- **消息编辑**：Discord 上编辑消息会再次触发处理，注意幂等，不然容易陷入循环刷工具调用。
- **身份映射**：同一个人在两个平台是两个 ID，session 天然不互通。需要打通就维护一张显式映射表放进工作区，别让模型"猜"这是同一个人。
- **自托管网络**：Telegram 走 long polling 即可，不必开 webhook；Discord 走 websocket 出站，注意防火墙规则。

## 可复用建议

- 大脑保持平台无关：提示词、记忆、MCP 工具全放 workspace，渠道只做薄适配。以后接第三个平台，只是加一段配置的事。
- 入站消息先归一化成统一结构（发送者、渠道、线程、附件），再进 Agent，工具和提示词就不用关心来源。
- 每渠道独立 allowlist 和独立日志标签，出问题十秒定位是哪一端。
- 先在两端各开一个测试群跑一周，观察 token 消耗和误触发，再放开真实场景。

## 总结

OpenClaw 把协议层差异基本抹平了，双渠道真正要自己设计的只有两件事：会话边界和身份映射。这两点想清楚，剩下的就是配置工作。一次部署、一个大脑、两个入口，比维护两套 bot 省心得多。有类似多渠道需求的可以照这个思路落地，欢迎在社区帖里补充你们的会话策略。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/c0c4d9d89fd7032b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/1ad2f2afd436dcdc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/0dcc920be0c55536.png)

