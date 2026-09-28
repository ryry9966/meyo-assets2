---
title: 一个 Agent 同时服务 Telegram 和 Discord：路由设计与踩坑记录
feedId: 39299
source: 综合讨论
publishedAt: 2026-09-28
---

## 背景

最初 Agent 只挂在 Telegram 上，服务个人和小群。后来团队协作搬到了 Discord，问题来了：再起一个实例，模型配置、MCP 工具、记忆文件都得维护两份，行为还会慢慢漂移。更合理的做法是：一个 Agent 内核，同时接两个通道。

## 问题

跨平台不是"多填一个 bot token"那么简单，核心是四件事：

1. **会话路由**：两个平台的消息如何映射到会话？同一个人在两边发言，算一个会话还是两个？
2. **格式差异**：Telegram 的 MarkdownV2 转义和 Discord 的 Markdown 是两套方言，长度上限也不同（4096 vs 2000）。
3. **主动消息**：定时任务、告警这类 Agent 主动发起的消息，该发回哪个平台？
4. **权限与身份**：两个平台各自的 allowlist 怎么管。

## 做法

### 单实例多适配器

网关保持一个实例，`channels` 下同时配 telegram 和 discord。内核只处理规范化信封：`{channel, chat_id, user_id, text, media}`。适配器只做两件事——入站翻译成信封，出站翻译回平台方言。适配层越笨越好，别把业务逻辑塞进去。

### 会话路由

会话键用 `platform:chatId`。我的选择是：同一人跨平台**不合并会话**，各自独立，但共享同一份工作区记忆（notes / 长期记忆通过 MCP 工具读写）。这样避免两个平台的消息流互相穿插导致上下文污染，又保证"Agent 记得你"的体验一致。真需要跨设备接续时，再显式做身份映射。

### 格式与长度

出站统一先产出内部格式，适配器负责转换：Telegram 侧用 HTML parse mode（MarkdownV2 的转义是重灾区）；Discord 侧超 2000 字符就切片或转文件上传。长文、代码块、附件都在这层消化，内核完全不感知。

### 主动消息的路由

定时任务和告警在创建时就绑定路由：日报发 Telegram 私聊，构建告警发 Discord 某个频道。路由绑定写在配置里，不写死在代码。每个 cron 任务记录 target（channel + chatId），触发时按此投递。

## 踩坑点

- **Discord Message Content Intent**：必须在开发者后台手动打开，否则机器人收到的消息内容全为空，现象极具迷惑性。
- **Telegram MarkdownV2 转义**：`_ * [ ] ( ) ~ > # + - = | { } . !` 全要转，漏一个就 400。直接换 HTML 模式省心。
- **自我回环**：群里如果还有别的 bot，按 sender id 过滤，避免 bot 互相触发死循环。
- **Discord 频道限速**约 5 条/5 秒，批量告警记得合并，否则会被静默丢弃。
- **时区**：两个平台分别配置，否则日报时间对不上。

## 可复用建议

- 信封模型 + 笨适配器，是将来加第三个平台（Slack、微信）成本最低的结构。
- 会话键宁可分裂也不要合并，记忆共享交给文件 / MCP 层。
- 每个通道单独做 ping / health 检查，出问题时先定位是通道还是内核。
- 日志统一带 channel 标签，排障时一眼分清流量来源。

## 总结

一个 Agent 服务多平台，关键不在"接"，而在"路由"：会话怎么分、格式谁来转、主动消息去哪。把这三件事在架构层定清楚，适配层保持无状态、无逻辑，后面加新平台基本就是复制粘贴级别的工。这套结构目前稳定跑了一段时间，两个平台行为一致，维护成本只有一份配置。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/7c4214b954350fda.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/e1f03daf4ab3116d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/ecc03e5a85173d48.png)

