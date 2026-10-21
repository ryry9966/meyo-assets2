---
title: 跨平台消息路由：让一个 Agent 同时值班 Telegram 和 Discord
feedId: 42157
source: 综合讨论
publishedAt: 2026-10-21
---

# 背景与问题

我们的社群一半在 Telegram、一半在 Discord，早期是两个 bot 各挂一个独立 Agent 实例。结果很典型：同一个问题 TG 答一遍、DC 再答一遍，答案还经常不一致；工具调用、长期记忆、限流预算全部翻倍。后来我们把两个平台收敛到**一个 Agent 核心**，前面各挂一个轻量 adapter，才算真正解决。

直接合并会遇到三类冲突：

1. **消息结构不同**：reply、thread、附件、mention 的表达方式完全不一样；
2. **会话粒度不同**：DC 按 channel/thread 切，TG 按 chat 切；
3. **回复格式不同**：markdown 子集差异、长度上限、转义规则各管各的。

# 做法

**1. 归一化信封（Envelope）**
定义统一结构：`platform / chat_kind / chat_id / user_id / reply_to / attachments`。两个 adapter 只负责收发和翻译，不含任何业务逻辑。

**2. 会话键设计**
用 `{platform}:{chat_kind}:{id}`，比如 `dc:thread:918`、`tg:group:-100xxx`。Agent 核心按会话键隔离上下文，平台之间默认不共享——除非显式做了用户身份映射。

**3. 单一输出、多端渲染**
Agent 只产出标准 markdown，各 adapter 自带 renderer：TG 侧走 HTML parse mode + 白名单标签，DC 侧直接透传；超长消息按段落切块，代码块绝不从中缝切断。

**4. 工具与记忆挂核心，不挂 adapter**
MCP server、检索工具、长期记忆都注册在 Agent 核心，两个平台天然共享同一套能力，token 预算也能统一管控。

# 踩坑点

- **Telegram MarkdownV2 转义是黑洞**，直接换 HTML parse mode，只留几个标签，砍掉 90% 的坑；
- **Discord 频道限流比想象中紧**，主动回复必须过发送队列，否则群发公告直接 429；
- **webhook 重试导致重复回复**，用 message_id 做幂等键去重；
- **附件上限不对等**，超过平台限制就转对象存储回链接；
- **断线恢复**：DC gateway 要处理 resume，TG getUpdates 要维护 offset，否则重启后重放旧消息；
- **最隐蔽的一类：thread/reply 映射错误**，导致 A 平台上下文串到 B 平台。上线前务必写一条“跨平台串扰”回归用例。

# 可复用建议

- adapter 保持“哑”，所有判断都在核心；
- Envelope 加 `version` 字段，后续接 Slack/QQ 不用动核心；
- 灰度路径：双平台只读观察一周 → 单群开放回复 → 全量；
- 监控按 platform 维度拆分延迟、错误率、token 消耗，否则出问题分不清哪端。

# 总结

跨平台路由的本质不是“接两个 API”，而是把**消息结构、会话模型、输出格式**三层差异全部收进 adapter/renderer，让 Agent 核心对平台无感。核心路由两百行以内就能写完，代码量不大，但会话键和幂等这些设计决策值得先想清楚再动手。做完之后最大的收益是行为一致：两个社群问同一个问题，得到同一个答案，用的是同一份记忆和工具。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/4f6c451d90066e47.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/097a4ec9ad4f63a3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/a51b2321252106aa.png)

