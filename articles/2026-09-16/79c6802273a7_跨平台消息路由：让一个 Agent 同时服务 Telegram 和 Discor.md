---
title: 跨平台消息路由：让一个 Agent 同时服务 Telegram 和 Discord
feedId: 37858
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

我们的用户群一半在 Telegram，一半在 Discord。之前维护两个 bot、两套 prompt、两份记忆，同一个问题在两边得到不同答案，运维上也要盯两套告警。这次重构的目标很明确：**Agent 核心只有一份，Telegram 和 Discord 只是两个不同的接入层。**

## 问题拆解

真正要解决的其实是四件事：

1. **消息模型不一致**：Telegram 的 update 结构和 Discord 的 gateway event 完全不同；
2. **会话隔离**：两边 chat 的粒度不一样（TG 是 chat_id，DC 还有 channel/thread 两层）；
3. **输出渲染差异**：双方 markdown 方言不兼容，消息长度限制也不同；
4. **稳定性**：限流规则、断线重连逻辑各一套。

## 做法

架构上只做一层抽象，不过度设计：

**1. 定义 PlatformAdapter 接口**

每个平台实现三个方法：`on_message`（拉消息）、`send`（发消息）、`format`（渲染文本）。核心 Agent 只面向统一消息事件：`platform / chat_id / user_id / text / reply_to / attachments`。

**2. 会话键设计**

`session_key = platform:chat_id`，Discord 下 thread 单独成键（`discord:channel_id:thread_id`）。同一个用户在 TG 和 DC 是两个 session，不做身份打通——没有可靠的映射依据，强行合并只会串数据。

**3. 渲染层降级**

Agent 输出统一的简化 markdown，渲染层按平台转换：TG 走 HTML parse mode（后面说为什么），DC 用原生 markdown。转换不了的结构化内容（比如 DC 的 embed）一律降级为纯文本，宁丑不炸。

**4. 出站队列 + 限速**

所有回复先进队列，按平台各自的限速规则消费。TG 按 chat 维度 1 msg/s，DC 按 webhook/bot 全局桶。长回复自动分段到各自上限以内。

**5. 能力层平台无关**

工具调用全部走 MCP 挂载（检索、DB 查询、定时任务），Adapter 不碰业务逻辑。换平台加功能时只改一处。

## 踩坑点

- **Telegram MarkdownV2 转义是地狱**。代码块外的 `_ * [ ]` 全要转义，漏一个就 400。直接切 HTML parse mode，只处理 `<` `>` `&`，问题消失。
- **Discord 忘开 `message_content` intent**，bot 能上线但收不到正文，排查了半天是 Portal 设置问题。
- **bot 互相触发死循环**。另一个 bot 的消息也会进来。规则：忽略一切 `from.is_bot = true` 的消息和自己发的消息，没有例外。
- **消息长度**：TG 上限 4096，DC 2000。不要硬切代码块，按代码块边界切分，否则渲染直接坏掉。
- **重连后的重复消费**：网关重连会重放事件，用 `message_id` 做幂等键，Redis SETNX 去重即可。

## 可复用建议

1. **先只读跑一周**：新平台先只记录不回复，观察消息分布和异常，再放开写入；
2. **日志统一带 `platform` 标签**：排障时一条 grep 就能分流两个平台的问题；
3. **不要做平台特性炫技**：DC 的 slash command、TG 的 inline keyboard 这类强绑定能力，只在你真的需要时做，且必须有无损降级；
4. **身份映射别硬做**：如果确实要打通用户，让用户主动 bind，不要靠用户名猜。

## 总结

这次重构的结论一句话：**平台差异全部关在 Adapter 里，核心 Agent 只认统一消息事件。** 总代码量反而比维护两套 bot 少了约四成，新增平台（比如之后的 Slack）理论上只需要再写一个 Adapter。消息路由这件事，难点从来不在“连上”，而在统一模型和边界条件的处理上。欢迎在群里交流你们的接入方案。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/9b24c22e2405f027.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/5cb6be333e3ea32d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/ad60fe3e3b639123.png)

