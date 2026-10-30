---
title: 一个大脑，两条通道：让同一个 Agent 同时服务 Telegram 和 Discord
feedId: 43001
source: 综合讨论
publishedAt: 2026-10-30
---

## 背景

我在 OpenClaw 上跑的这个 Agent，最初只挂了 Telegram，用于私聊和一个小群。后来协作讨论迁到了 Discord，问题就来了：再起一个实例？那意味着两份会话记忆、两套配置、两倍维护成本。更实际的做法是单实例多通道——一个大脑，两条入口。

## 问题

把两个通道接起来，难点不在“连上”，而在三个层面的差异：

- **会话模型**：Telegram 靠 `chat_id`，Discord 是 channel 之下还有 thread，路由键不一样，处理不好会出现 thread 串台。
- **消息方言**：MarkdownV2 与 Discord markdown 不兼容，长度上限一个 4096 一个 2000，引用/回复的表示完全不同。
- **身份与限流**：两边 user 体系不互通，限流策略也不同，直接透传很容易吃 429。

## 做法

核心原则一句话：**适配器保持“哑”，一切策略收在 core。**

1. 每个通道只写一个薄适配器，负责收发和原始 payload 透传。
2. 定义统一消息信封：`channel / chat_id / thread_id / user_ref / media / raw`，其中 `raw` 原样保留，排障时能对账。
3. 会话路由键取 `{channel}:{chat_id}:{thread_id}`，Discord 的 thread 天然隔离。
4. Agent 输出统一的 Markdown 安全子集，出站由各通道的 formatter 插件做二次渲染和转义。
5. 出站统一走限速队列，按通道上限切块（我留了余量：3800 / 1900）。
6. 身份映射表做成可选项：把 TG 和 Discord 的账号关联到同一 persona 以共享记忆，默认关闭，避免误合并。

配置大致长这样：

```yaml
channels:
  telegram: { mode: polling }
  discord:  { intents: [guild_messages, message_content] }
router:
  session_key: "{channel}:{chat_id}:{thread_id}"
  chunk: { telegram: 3800, discord: 1900 }
  outbound_queue: rate_limited
```

## 踩坑点

- **Discord 收不到正文**：MESSAGE CONTENT intent 默认关闭，现象是“看得到有人说话，内容为空”，第一次接很容易当成 bug 查半天。
- **TG 群里静默**：privacy mode 下 bot 收不到普通消息，要么关 privacy，要么给管理员。
- **MarkdownV2 转义**是花时间最多的坑，下划线和代码块尤甚；我的解法是 formatter 只产出安全子集，宁可朴素。
- **分块后的引用噪音**：Discord 里每块都带 reply 引用会很吵，只在第一块带。
- **幂等**：轮询重试会造成重复投递，去重键用通道消息 id，不要用时间戳。
- **不要互为桥**：两个通道的回复互不转发，否则一不小心就回环刷屏。

## 可复用建议

- raw payload 必留，日志按通道打点（收/发/失败/429 分开计数），否则排障全靠猜。
- 新通道渐进放开：先只读观察一周，再开私聊，最后开群。
- formatter 做成插件而不是 if-else，后面接 Slack、微信只是加一个文件的事。

## 总结

多通道的意义不是省一台机器，而是让同一份记忆和人格贯穿所有入口。把平台差异消化在适配层，core 只面对统一信封——这套结构跑稳之后，再加第三个通道，工作量大概只有一个下午。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/ba1c3c6b05063ed4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/75b784258fac963d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/3553b4ed892f4916.png)

