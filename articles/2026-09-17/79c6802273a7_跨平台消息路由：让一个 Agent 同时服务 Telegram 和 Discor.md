---
title: 跨平台消息路由：让一个 Agent 同时服务 Telegram 和 Discord
feedId: 37989
source: 综合讨论
publishedAt: 2026-09-17
---

## 背景

我们的 Agent 最初只挂在 Telegram 上，服务一个技术讨论群。后来团队日常协作迁到了 Discord，需求变得很直接：两边的人都想用同一个 Agent。如果为了 Discord 再起一个独立实例，意味着记忆分裂、工具配置（MCP server）漂移、双份运维。所以目标是单实例双 channel：一份记忆、一套工具，同时服务两个平台。

## 问题

把 telegram 和 discord 两个 channel 直接挂到同一个 gateway 上，很快会撞上三类问题：

1. **会话串扰**：不同平台的消息如果落进同一个 session，上下文互相污染；
2. **格式方言**：Telegram 的 MarkdownV2 和 Discord 的 markdown 转义规则差异很大，透传必炸；
3. **平台限制不同**：消息长度上限、速率限制、线程与引用语义都不一样。

## 做法

核心思路一句话：**隔离做在 session 键上，适配做在渲染层**。

1. **单实例双 channel**：gateway 同时启用两个 channel，各填各的 bot token，Agent 进程只跑一个；
2. **会话键隔离**：session key 按 `channel:chat-id` 维度切分，Telegram 群和 Discord 频道天然是不同 session。同一用户跨平台共享记忆的场景，单独维护一张 alias 映射表；
3. **统一中间表示**：入站消息统一归一化成内部格式——纯文本、附件引用、回复引用、sender 元数据。Agent 只面对这一种格式；
4. **渲染适配器**：出站写两个函数 `renderTelegram` / `renderDiscord`，内部逻辑只产出一份 Markdown，各渲染器负责自己的转义和长度截断；
5. **限速与分段**：发送前按平台上限切分（Telegram 4096、Discord 2000），出站统一走队列，配合 typing 状态避免用户以为卡死。

## 踩坑点

- **MarkdownV2 转义**：`_` `.` `[]()` 都要转义，我们一开始直接透传输出，代码里带下划线就报 400。最后统一策略：先保护代码块，再做字符级转义；
- **线程语义**：Discord 的线程内回复必须显式带 thread id，否则消息会飞到主频道；Telegram 的 reply_to 对应的是 message reference，两者不能混用一个“回复”抽象；
- **速率**：Discord 按 channel 限速，Agent 并发回复多个频道时很容易吃 429，出站队列 + 退避重试是必须的；
- **附件**：Discord CDN 链接带签名会过期，入站附件要先落地本地存储再交给 Agent，不能只存 URL；
- **身份映射**：同一个人在两个平台的 user id 完全不同，跨平台记忆共享别指望平台帮你，alias 表要自己维护。

## 可复用建议

1. 先抽象 ChannelAdapter 接口（normalize 入站 / render 出站），以后接新平台只需实现两个函数；
2. 出站队列顺带解决了幂等问题：重试时用同一 message id 去重；
3. 调试用干跑模式：渲染结果只打日志不实际发送，肉眼比对两个平台的输出差异；
4. session 键命名规范化，把 channel、chat id、thread 都写进 key，排查串扰时日志能直接对上。

## 总结

跨平台路由不是把 token 填两份就完事，真正的工作量在会话隔离、格式适配、平台限制这三件事上。把隔离收敛到 session 键、把适配收敛到渲染层，一个 Agent 服务多平台是完全稳定的。我们这套结构跑了一个多月，两边日均几百条消息，没有再出过串扰和格式事故。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/36b978f0a8d07a3d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/28d732e8c49216ab.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/ca81a736c52666fa.png)

