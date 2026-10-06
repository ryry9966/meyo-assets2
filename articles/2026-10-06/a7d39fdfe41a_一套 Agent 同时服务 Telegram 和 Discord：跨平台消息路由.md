---
title: 一套 Agent 同时服务 Telegram 和 Discord：跨平台消息路由实践与踩坑
feedId: 40689
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

我们社区的用户一半在 Telegram、一半在 Discord。早期图省事，跑了两套独立的 Agent 实例：各自的配置、各自的记忆、各自的插件目录。结果是同一个问题两边答案不一致，维护成本翻倍。目标很朴素：**一套核心（模型、工具、记忆、MCP 服务），两个入口。**

## 问题

把两个 channel 直接挂上同一个 Agent，并不等于"跨平台"，立刻会遇到：

- **会话隔离**：Telegram 的 `chat_id` 和 Discord 的 `channel_id` 是两个不相交的 ID 空间，session key 设计不当会互相污染或彻底分裂；
- **平台差异**：Discord 单条消息 2000 字符，Telegram 是 4096；两边 Markdown 方言不同；触发方式也不同（TG 群里靠 @，Discord 靠 mention/斜杠命令）；
- **限流**：Discord 的 rate limit 更细，长回复一股脑发出去就是 429；
- **运维**：两个 gateway 进程日志混在一起，重连行为还不一致。

## 做法

核心原则一句话：**Adapter 只做翻译，路由决策放在核心层。**

1. **统一入站模型**。每个 channel 适配器把原始事件归一化为 `{platform, chat_id, user_id, text, msg_id}`，`msg_id` 加平台前缀（`tg:xxx` / `dc:xxx`），作为幂等键。
2. **Session key 用 `platform:chat_id`**。需要跨平台共享的知识（项目文档、常用命令）不放 session，放 MCP 记忆工具里，session 只保留各自的对话上下文。
3. **出站走统一 formatter**。核心层只产出纯文本和块结构，拆分、转义、格式转换全部下放到各 adapter：Discord 按代码块边界分段，Telegram 用 HTML parse_mode，实测比 Markdown 模式稳。
4. **配置上两个 channel 指向同一个 workspace**，allowlist 各自维护：

```yaml
channels:
  telegram:
    allowFrom: [...]
  discord:
    guilds: [...]
agents:
  main:
    workspace: ./shared-ws
```

5. **出站队列按 platform 分桶**，Discord 桶加 500ms 间隔，超长消息分片发送。

## 踩坑点

- **最先翻车的是 Markdown**：同一段回复在 TG 正常、在 Discord 因为代码块嵌套直接炸格式。结论是核心层严禁吐平台语法，formatter 各自实现。
- **Discord 重连会重放事件**：没做幂等之前，一条消息触发两次工具调用。靠带前缀的 `msg_id` 去重解决。
- **TG 的 privacy 模式**：群聊默认只收 /命令和 @mention，排查了半天"路由坏了"，其实消息根本没进网关。
- **分段不能按字符数硬切**：要按代码块和段落边界切，否则切在中间照样报错。

## 可复用建议

- Adapter 保持"薄"，所有平台特性在 adapter 内消化；
- 入站先归一化再进核心，幂等键从一开始就设计好；
- 会话按 `platform:chat` 隔离，跨平台知识下沉到记忆层；
- 上线顺序：先 Discord 测试服灰度跑一周，再切 TG 主群。

## 总结

跨平台不是"多接一个 channel"，而是把**翻译**和**决策**拆干净。adapter 薄、核心纯、队列分桶，这三点立住了，后面接 Slack 或飞书，也只是再多写一个 adapter 的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/d747fe3a10c09efa.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/d0c5ee828c8a354b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/ca0276b43bf241e0.png)

