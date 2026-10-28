---
title: 一个 Agent 同时服务 Telegram 和 Discord：跨平台消息路由实践
feedId: 42858
source: 综合讨论
publishedAt: 2026-10-28
---

## 背景

我平时私聊走 Telegram，社区协作在 Discord。之前是两个 bot 各连一个 Agent 实例，记忆不同步、配置要维护两份，同一个问题在两边经常得到不同的回答。这次把它们收敛到同一个 OpenClaw 实例：两个平台只是不同的入口，大脑只有一个。

## 问题

核心矛盾有三个：

1. **接入**：两套协议（Telegram Bot API 长轮询 / Discord Gateway WebSocket）如何汇入同一个 agent 循环；
2. **路由与会话**：哪些对话共享上下文，哪些必须隔离；
3. **出站差异**：长度上限、Markdown 方言、回复/线程语义都不一样，不能让模型层去操心这些。

## 做法

我的拓扑是「薄适配层 + 统一事件总线 + 单 agent 循环」：

1. **两个 channel adapter 独立运行**，只负责收发和协议转换，把消息归一化成内部事件：`{platform, chat_id, user_id, kind, payload, reply_to}`。
2. **session key 设计成 `platform:chat_id`**，默认完全隔离，避免两边上下文互相污染。再用 identity mapping 把我自己的 Telegram user id 和 Discord user id 绑到同一个 owner，挂一个共享的长期记忆空间——会话隔离，但"我是谁、我的偏好"是全局的。
3. **路由规则集中在一处配置**：Telegram 私聊直达主 agent；Discord 服务器里只有 @ 提到 bot 或指定频道的消息才进队列，其余忽略。
4. **出站渲染按平台走**：适配层负责翻译 Markdown 方言——Discord 侧做 2000 字符分片，Telegram 侧处理 4096 上限和 parse mode；图片、语音先落本地缓存，再把路径交给 agent。
5. **防回环**：出站消息带来源标记，适配层直接丢弃 author 是自己的事件。

## 踩坑点

- **Telegram MarkdownV2 转义是重灾区**：代码块里的 `_`、`.`、`!` 不转义直接 400 报错。我最后改成 HTML parse mode，省掉大量正则修补。
- **Discord 限流**：多频道同时活跃时很快撞 429，务必读响应里的 Retry-After 排队重试，别用固定 sleep。
- **会话串台**：最早偷懒用全局 session，结果 Telegram 里的调试上下文泄漏进了 Discord 回复。教训：隔离必须是默认值，共享要显式声明。
- **重连重放**：Discord 断线重连后会重放事件，没有 author 过滤时 bot 差点自己跟自己聊起来。同理，别把其他 bot 的消息也喂进 agent。

## 可复用建议

- 适配层做薄，路由、限流、分片这些"聪明"逻辑放在总线或 agent 侧，换平台时只重写薄壳。
- 内部事件格式先定死再动手，之后加平台才是"加一个 adapter"的事。
- 上线前先接一个回声频道，所有出站消息先进测试频道观察几天。
- 每个平台独立限流器和重连退避，不要让一个平台的高峰拖垮另一个。

## 总结

跨平台路由的难点不在"连上"，而在会话边界和出站语义的归一化。把协议差异挡在薄适配层里、路由规则集中管理、隔离作为默认，一个 Agent 服务多个渠道就是很平常的工程问题。整个改造大概两三百行代码，收益是配置减半、记忆统一、两边回答一致。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/345595a674c79de4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/448a0bf4fca55e25.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/651b4bb9d0a90460.png)

