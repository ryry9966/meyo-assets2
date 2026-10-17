---
title: 跨平台消息路由：让一个 Agent 同时值守 Telegram 和 Discord
feedId: 41843
source: 综合讨论
publishedAt: 2026-10-18
---

## 背景

我的 Agent 最初只挂在 Telegram 上，服务自己和几个同事。后来团队协作迁到了 Discord，需求变成：同一份知识库、同一套 MCP 工具，但两个入口都能找到它。第一反应是再起一个实例，很快放弃——两份上下文、两份配置、两处排障，长期维护成本太高。最后走的是单 gateway 多 channel 的路线。

## 问题

真正要解决的只有三件事：

1. **会话边界**：两个平台的消息不能互相串台；
2. **格式差异**：Discord 和 Telegram 对 Markdown、长度上限、媒体的处理各不相同；
3. **触发规则**：私聊、群聊、@提及的策略不一样。

## 做法

**1. 单 gateway 挂双 channel。** 一个 OpenClaw gateway 同时配置 telegram 和 discord 两个 channel，各填各的 bot token 和 allowlist。进程一份，模型调用一份，成本没有翻倍。

**2. 会话按 channel 隔离。** session key 带 channel 维度，Telegram 用户一个 session，Discord 频道一个 session。需要共享的只有长期记忆：放进 workspace 的笔记文件，让 Agent 需要时自己去查，而不是靠上下文硬记。

**3. 把渠道能力差异写进系统提示。** 明确告诉模型：当前在哪个平台、Discord 单条 2000 字符、Telegram 4096、表格只在 Discord 渲染、代码块两边都行。就几行字，能避免一大半格式事故。

**4. 跨平台推送走工具，不走路由。** 路由只回答"这条消息归谁回"。想让 Agent 把提醒推到另一个平台，封装一个 notify 类的 MCP 工具，参数里显式指定目标 channel，行为可审计。

**5. 灰度开启。** 先只开私聊跑一周，稳定后再放开 Discord 服务器里的 @提及触发。

## 踩坑点

- 早期偷懒让两个平台共用 session，结果 Discord 里的讨论出现在 Telegram 的回复里，立刻改回隔离。串台比不回复更伤信任。
- Discord 开发者后台没开 Message Content Intent，bot 收到的消息全是空的。排查半小时才发现是平台侧开关，不是配置问题。
- 长回复在 Discord 被静默截断。后来在提示里要求"先给结论再分条"，超长内容配合工具分段发送。
- Telegram 侧 parse_mode 不匹配时表现为发送失败且报错隐晦，统一降级为纯文本最省心。
- Discord 速率限制按桶计算，Agent 一次推十几条提醒会触发等待，改成聚合后单次发送。

## 可复用建议

- channel 适配层保持薄，业务逻辑全收在 Agent 工具层，换平台只是换皮；
- 日志带 channel 前缀，一眼定位问题在哪个入口；
- 能力差异写在提示词里而不是硬编码，调整成本最低；
- 群聊触发策略宁可先紧后松。

## 总结

一个 Agent 服务两个平台，九成的工程量在"边界"：会话边界、格式边界、触发边界。路由本身反而简单。把这三条边界用配置和提示词显式表达出来，之后接入第三个平台，基本只是再加一份 channel 配置的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/90a0ef7a053e2bc1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/46b863904a9e6c45.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/dc3716c27a629b2a.png)

