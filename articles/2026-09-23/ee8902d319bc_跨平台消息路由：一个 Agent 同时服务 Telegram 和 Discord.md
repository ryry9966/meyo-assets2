---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 38583
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

我们团队基于 OpenClaw 跑着一个内部 Agent，挂着几个 MCP 工具（文档检索、工单查询、日程变更），最初只接了 Telegram。后来一部分协作用户迁到 Discord，需求很直接：同一个 Agent、同一份记忆，两边都能用。

## 问题

第一版做法很土：复制一份配置，再起一个 bot 进程，两边各自直连 Agent。跑了两周暴露出三个问题：

1. **上下文分裂**：同一用户在 Telegram 问过背景，到 Discord 再问，Agent 完全失忆；
2. **维护双份**：提示词、工具白名单改一次要同步两处，迟早出错；
3. **平台差异渗透**：消息长度上限、Markdown 方言、回复引用格式都不一样，Agent 侧到处是 if/else。

## 做法

重构的核心思路是把"平台"和"智能"彻底拆开：

1. **定义统一消息信封**。所有入站消息先转成信封：`platform`、`user_id`、`channel_id`、`reply_to`、`attachments`、`capabilities`（该平台是否支持按钮、线程、长文）。
2. **每个平台一个薄 adapter**。`telegram-adapter` 和 `discord-adapter` 只做两件事：入站转信封；出站把 Agent 的中性 Markdown 转成平台方言（Telegram 用 HTML parse mode，Discord 用自己的 markdown）。
3. **路由层做会话决策**。默认按 platform+user 隔离上下文；用户可发绑定命令把两个平台的 ID 显式关联，关联后共享会话记忆。绑定必须二次确认，不做自动合并。
4. **Agent 与 MCP 保持平台无关**。系统提示词、工具注册只有一份，路由层剥掉信封后喂给 Agent，输出只含中性 Markdown 和建议动作。
5. **公共能力上提**：限流、重试、message id 去重、typing 状态全部放在路由层，adapter 里不允许出现这些逻辑。

部署上是一个 core 进程加两个 adapter 进程，信封走 Redis 队列。这样 Telegram 侧故障不会拖垮 Discord。

## 踩坑点

- **长度分片**：Telegram 上限 4096，Discord 是 2000。分片在出站 adapter 按"目标上限留余量"处理，且分片点优先选代码块边界，否则截断后的代码块很难看。
- **Discord 3 秒 ACK**：交互必须先 acknowledge 再异步计算，否则前端直接报"应用无响应"。Telegram 没这个约束，所以 ACK 逻辑只能各自留在 adapter。
- **Telegram webhook 会重复投递**：必须按 message_id 做幂等去重，否则一条消息触发两次工具调用。
- **转义是重灾区**：Telegram HTML 模式下未转义的 `<` 出过解析失败，Discord 里裸 `@everyone` 意外 ping 了全频道。出站转换要白名单化，不做裸透传。
- **身份绑定别图省事**：我们试过按用户名自动匹配，撞名导致把两个人的会话合并了。回滚后改成显式命令加确认。

## 可复用建议

- **信封模式是关键**。后面接 Slack、飞书，只需再写一个 adapter，核心一行不改；
- **capabilities 字段值得做**：Agent 检测到"该平台不支持按钮"时可自动降级为纯文本选项，而不是报错；
- **adapter 保持愚蠢**：限流、幂等放路由层，adapter 越简单越不容易各自腐化；
- **灰度上线**：新平台用 feature flag 控制，先只对内部频道开放，观察一周再放开。

## 总结

这次重构最大的收益不是省了一个 bot 进程，而是让 Agent 彻底平台无关：提示词、记忆、工具只有一份，平台差异全部下沉到几百行一个的 adapter 里。经验浓缩成一句话：**路由层做薄，信封做稳，身份绑定做保守**。下一个要接的入口大概率还是这套结构，边际成本应该只剩写 adapter 本身。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/7ae04b3636c199f6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/6b1f74179ddf740c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/2ab271335f14c2b1.png)

