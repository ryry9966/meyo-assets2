---
title: 一个 Agent 同时服务 Telegram 和 Discord：跨平台消息路由实践
feedId: 38350
source: 综合讨论
publishedAt: 2026-09-21
---

## 背景

我日常在 Telegram 群里处理个人事务，团队协作和社区讨论则都在 Discord。之前给 OpenClaw 只配了 Telegram 单通道，Discord 侧的问题要么自己复述一遍，要么在两边各养一个 Bot、各维护一份配置。知识库更新一次要同步两处，一周后就开始不一致了。这次目标很直接：**一个 Agent、一份工作区，两个通道都能进。**

## 问题

难点不是"能不能接两个通道"，而是三件事：

1. **会话隔离还是共享**：TG 群和 Discord 频道的上下文要不要打通？
2. **触发规则**：群聊不能每条都回，两个平台的 mention/回复语义还各不相同。
3. **格式差异**：Discord 的 markdown 方言和 2000 字符限制，Telegram 的 MarkdownV2 和 4096 限制，按钮/组件互不兼容。

## 做法

结构上让一个 gateway 同时挂两个通道，Agent 只保留一个实例：

1. **通道接入**：gateway 配置里同时声明 `telegram` 和 `discord` 两个 channel，各填各的 bot token。注意 Discord Bot 要开 `MESSAGE CONTENT INTENT`，否则收不到消息正文。
2. **会话键设计**：session key 采用 `agent:channel:chatId` 的形式，把 channel 写进 key，避免不同平台的 chat ID 撞车。默认各群隔离。
3. **共享状态走 MCP**：跨通道需要的记忆、待办、长期笔记统一放进一个 memory MCP server，通道无关。于是 TG 里让 Agent 记的事，Discord 侧问一句它能答上来——上下文不共享，但"事实"共享。
4. **触发规则**：群聊只响应 @mention、回复 Bot 的消息和少量白名单关键词；私聊全响应。规则写在配置里，不靠 prompt 硬扛。
5. **格式适配**：回复默认走纯文本加基础 markdown，超长内容分段发送；把"当前所在通道"作为上下文字段注入 prompt，Agent 自行调整口吻和长度。

## 踩坑点

- **双 Bot 互吹**：最初在两边做了个简单消息同步，结果两个 Bot 对同步消息各自回复，刷了三页才停。解法：忽略一切来自 bot 的消息，且只响应 @ 自己的。
- **长度截断**：Discord 的 2000 字符是硬限制，长回复直接发送失败。现在统一按 1800 字符分块，TG 侧也控制在 3800 以内。
- **按钮不通用**：TG 的 inline keyboard 和 Discord components 是两套体系。涉及确认的流程我改成了纯文本"确认/取消"，牺牲一点体验换可移植性。
- **工作区文件锁**：不要起两个 gateway 实例共享同一 workspace、各自挂一个通道，memory 文件会写花。一个 gateway 挂两个 channel 足够。
- **日志带通道标签**：排障时 `[tg:xxx]` / `[dc:xxx]` 前缀能省一半时间。

## 可复用建议

- **路由层保持"笨"**：触发、限流、分块、格式转换全部做成确定性逻辑，不要让 LLM 决定"要不要回复"。
- **知识和记忆放在通道无关的位置**（workspace 文件或 memory MCP），通道配置里只放凭证和触发规则。
- **上线前先拿一个私聊测试群跑一周**，重点观察触发边界和分块表现，再放开到正式群。

## 总结

一个 Agent 服务多平台，核心是把"路由"和"智能"分开：路由层负责通道接入、触发判断和格式这些确定性工作，Agent 只面对一份工作区。跑了一个月下来，两边维护成本基本归一，知识库只有一份，心智负担小了很多。如果你也在两个平台之间反复横跳，这个结构值得直接抄。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/8d9eab41bac7b4e0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/f4c32bd109f7b09b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/f788e0baddfc85d2.png)

