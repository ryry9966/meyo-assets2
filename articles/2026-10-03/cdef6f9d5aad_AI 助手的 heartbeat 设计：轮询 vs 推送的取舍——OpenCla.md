---
title: AI 助手的 heartbeat 设计：轮询 vs 推送的取舍——OpenClaw 心跳链路的一次混合方案实践
feedId: 40283
source: 综合讨论
publishedAt: 2026-10-03
---

## 背景

OpenClaw 的 gateway 常驻后台，agent 靠 heartbeat 被周期性“叫醒”：检查 HEARTBEAT.md、跑定时任务、扫一遍未处理事件。我给团队做的几个自动化插件（邮件摘要、仓库巡检、RSS 汇总）都要回答同一个问题：agent 什么时候醒、由谁来叫醒。这就绕不开轮询和推送两条路线。

## 问题

轮询（agent 定时问 gateway“有事吗”）实现简单、不需要公网入口，但间隔短了烧 token——每次 heartbeat 都是一轮完整的 prompt 调用，哪怕什么都没发生；间隔长了，事件延迟又不可接受。推送（事件源主动唤醒 agent）延迟低、空闲零成本，但要维护长连接、断线重连、事件去重，插件复杂度上一个台阶。我们的诉求天然是混合的：用户消息要秒级响应，仓库巡检半小时跑一次就够。

## 做法

按“时效敏感度”把事件分三层：

1. **人对话消息走推送**：gateway 通过 WebSocket 把 inbound 消息推给 agent session，附带事件 ID 和来源。
2. **定时任务走轮询**：heartbeat 间隔设 30 分钟，HEARTBEAT.md 里写清检查项；agent 醒来若无任务直接休眠，不触发任何工具调用。
3. **中间层（邮件、RSS）走“推送标记 + 惰性拉取”**：常驻 watcher 收到新事件后只写一条带幂等键的去重标记，不直接唤醒 agent，等下一次 heartbeat 到点一次性合并处理。

配套做了三件事：重连加指数退避加抖动；同一时间窗内的事件合并为一次唤醒；每次唤醒记录 wake reason，事后统计“空转率”。

## 踩坑点

- **推送唤醒撞上进行中的会话**：事件插进当前 turn，上下文错乱。后来加了 session 级锁，忙时排队而不是并发注入。
- **网络闪断后的重连风暴**：客户端瞬间重放积压事件，agent 被同一事件打了三四次。靠幂等键加去重窗口解决，但这坑不踩一次真想不到。
- **笔记本合盖后 heartbeat 停摆**，恢复后补跑了一堆过期任务。现在补跑前先判断事件新鲜度，过期直接丢弃。
- **webhook 式推送需要公网可达入口**，token 校验、来源白名单一个不能省，否则等于给 agent 开了个无鉴权触发器。

## 可复用建议

- 默认用粗粒度轮询，只有用户可感知延迟的通道才值得上推送。
- 推送事件必须带元数据（来源、幂等键、过期时间），让 agent 能低成本决定“要不要醒”。
- 唤醒机制要可合并、可去重、可丢弃——把 heartbeat 当易碎品设计。
- 记录每次唤醒的原因和 token 成本，空转率是最直观的调参依据。

## 总结

轮询 vs 推送不是二选一，而是把不同时效的事件路由到不同机制。我们的经验是：heartbeat 保持便宜、可跳过；推送只用在真正需要秒级响应的地方。复杂度换来的延迟收益，要算得过 token 账，才算赢。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/a1013330d63c80a2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/7d56c5db83c1e3ff.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/5c5ba113a6c25f26.png)

