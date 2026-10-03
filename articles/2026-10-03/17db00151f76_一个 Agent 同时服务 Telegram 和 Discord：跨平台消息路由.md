---
title: 一个 Agent 同时服务 Telegram 和 Discord：跨平台消息路由实录
feedId: 40260
source: 综合讨论
publishedAt: 2026-10-03
---

## 背景

场景很典型：中文协作群在 Telegram，海外兴趣社区在 Discord。Agent 跑在家里的常驻机器上，OpenClaw gateway 之前只挂了 Telegram 单渠道。需求不复杂：两个平台都能 @ 它，而且它“认识”同一个人，不出现两套记忆、两份人格。

## 问题拆开是四个

1. **接入**：两个 bot token，两套事件模型；
2. **身份**：同一个人在两边的 user id 完全不同；
3. **会话**：session key 怎么定，跨平台共享还是隔离；
4. **出站**：渲染差异——Discord 的 2000 字符限制，Telegram 的 parse mode 和转义。

## 做法

**第一步：双渠道接入。** Telegram 走 BotFather 拿 token；Discord 建 Application 后，务必在 Developer Portal 打开 Message Content Intent。binding 指向同一个 agent：

```json
"bindings": [
  { "channel": "telegram", "target": "group:xxx", "agent": "main" },
  { "channel": "discord",  "target": "channel:yyy", "agent": "main" }
]
```

**第二步：身份归一。** 写了个小钩子，入站消息进 agent 前查一张映射表（SQLite 足够）：`(platform, user_id) -> internal_identity`。首次见到建匿名条目，人工补一次别名即可。这步成本极低，收益最大——agent 跨平台提到同一人不再“精分”。

**第三步：会话策略。** 默认按 `(channel, chat_id)` 隔离，避免两边上下文串台；确需共享的任务用显式 session key 走独立线程：Discord thread、Telegram 群 topics。体验上比共享主会话干净得多。

**第四步：出站适配层。** 定义统一中间消息模型，渲染时按平台降级：超长内容 Discord 按代码块边界分片（必要时转入线程），Telegram 一律 HTML parse mode，放弃 MarkdownV2。两个平台各挂出站队列，429 和频控统一退避重试。

## 踩坑点

- **Discord 收得到事件但 content 为空**：Intent 没开，Bot 权限页也要勾，改完重启生效。
- **MarkdownV2 转义地狱**：下划线、括号、句点全要转义，漏一个整条消息 400。换 HTML 后问题消失。
- **分片切断代码块**：先按 ``` 边界切，再按长度切，顺序反了必翻车。
- **重复入站**：Discord 网络层重试会让同一条消息处理两次，用 message id 做幂等键。
- **@提及解析**：Discord 是 `<@id>`，Telegram 是 entities，适配层必须先归一再进 agent，否则路由判断全部失灵。
- **频控**：Telegram 群组 20 条/分钟，Discord 按 bucket 限速，队列 + 指数退避是底线，别裸发。

## 可复用建议

1. 把平台差异全部关进入站/出站两个适配层，agent 层只面对统一消息模型。这条纪律守住，接第四、第五个平台是增量而不是重写。
2. 身份映射和 session key 的设计先于功能开发，事后补的代价是清历史记忆。
3. 灰度顺序：单平台双向跑稳 → 第二平台只读观察路由 → 再放开双向。
4. 每个 channel 的日志打独立 tag。排障时先分清是适配层问题还是 agent 层问题——一半的“agent 犯傻”其实是渲染层 400。

## 总结

跨平台路由没有黑魔法，OpenClaw 的 channel 机制完全够用。真正的工程量在两处：把交互差异收敛到边界适配层，以及身份与会话这两个数据模型问题。这一轮做完，再接新平台基本就是照模板填空。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/671d914cec9d686d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/31c3a5f277dbc799.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/4f2b29d341cafdb0.png)

