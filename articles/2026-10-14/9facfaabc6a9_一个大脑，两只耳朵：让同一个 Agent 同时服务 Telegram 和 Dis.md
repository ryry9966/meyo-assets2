---
title: 一个大脑，两只耳朵：让同一个 Agent 同时服务 Telegram 和 Discord
feedId: 41461
source: 综合讨论
publishedAt: 2026-10-14
---

## 背景

最早的场景很典型：Agent 只挂在 Telegram 群里给几个老朋友用，后来团队协作搬到了 Discord，又不好意思让所有人换个平台。第一反应是再部署一套——但很快意识到这是下策：两份 workspace、两份记忆、两份 prompt，人格会漂移，工具配置要同步两遍，出了 bug 排查两次。

所以目标收敛为一句话：**单实例 gateway、单份 Agent、双通道入口**。本文记录实际跑通的过程和踩的坑。

## 问题

表面上把 channels 配置加一段就完事，但真正要回答的是三个问题：

1. 两个平台的消息**会话如何隔离**，不会互相串上下文；
2. 回复**如何路由回来源**（而不是发错平台）；
3. heartbeat / cron 这类**主动消息**到底投递到哪里。

## 做法

**1. 单 gateway，双通道。** 不要起两个进程。配置里同时声明两个 channel，各用各的 bot token：

```json
{
  "channels": {
    "telegram": { "botToken": "..." },
    "discord": { "token": "..." }
  }
}
```

**2. 会话隔离靠 sessionKey 天然实现。** OpenClaw 的会话键按 `channel:chatId` 组织，Telegram 的 chat 和 Discord 的频道落在不同 namespace，天然不会串。这是能"一脑多用"的前提，不需要自己写隔离逻辑。

**3. 逻辑全部收在 Agent 层。** 人格 prompt、MCP 工具、插件只维护一份。平台差异不要下沉到业务逻辑里，只允许出现在两个地方：适配层（框架处理）和 system prompt 里的一小段（比如"Discord 里回复尽量短、优先用 thread"）。

**4. 主动消息显式指定通道。** heartbeat 和定时任务触发时没有"来路"可回，投递目标必须写清楚，否则会默认只发到其中一个通道，另一半用户永远收不到通知。

**5. 先最小链路联调。** 两端各建一个测试群，只接一个 ping 工具，确认收发、会话隔离、回复路由都正常，再接真实工具。

## 踩坑点

- **Discord 收不到消息正文**：多半是开发者后台没开 Message Content Intent，日志里表现为收到空消息。
- **Telegram 报 409**：说明有另一个进程在用同一个 token 拉 getUpdates，常见于旧实例没杀干净。
- **消息长度上限不同**：Discord 单条 2000 字符，Telegram 4096。长回复要么分段，要么让 Agent 落盘后发文件链接。
- **Markdown 方言不一致**：引用、剧透标签、代码块渲染两边行为有差异，格式化输出要收敛，别用花哨语法。
- **媒体路径不同**：Telegram 的 photo 和 Discord 的 attachment 进入 Agent 的方式不一样，多模态工具必须双端各测一遍。
- **身份不互通**：同一个人在两个平台是两个身份，白名单和权限要分别配置，别指望自动映射。

## 可复用建议

- **适配层保持薄**。所有逻辑收敛在 Agent + 工具层，以后加 Slack、加微信只是多一个 channel 配置，不动核心。
- **在 repo 里维护一张路由表**：哪个群/频道 → 哪个 Agent → 哪个 workspace。三个月后你自己会感谢这张表。
- **排查问题时按 channel 前缀过滤日志**，两条链路分开看，比整体 grep 快得多。
- 跨平台的主动消息单独建一个 heartbeat 场景来测，不要混在被动回复链路里验证。

## 总结

跨平台服务不是复制 Agent，而是**共享一个大脑，接多对耳朵和嘴巴**。OpenClaw 的 session 隔离和 channel 抽象基本够用，真正的工作量都在平台细节上：intent、长度上限、格式差异、身份体系。建议的路径是先跑通 ping 级最小链路，确认路由正确，再逐步把真实工具和主动消息迁上来。整个过程没有黑魔法，只是把"平台差异"老老实实地关在适配层里。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/728f7ae1955d2984.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/0752cdb5ad837ce9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/e1a176518d96999b.png)

