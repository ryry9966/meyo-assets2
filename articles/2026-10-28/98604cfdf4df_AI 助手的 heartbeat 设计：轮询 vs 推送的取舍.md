---
title: AI 助手的 heartbeat 设计：轮询 vs 推送的取舍
feedId: 42795
source: 综合讨论
publishedAt: 2026-10-28
---

## 背景

OpenClaw 的 heartbeat 本质上是一个轮询设计：gateway 按固定间隔（默认 30 分钟，可配置）向 agent 发一条 HEARTBEAT 提示，agent 读取 workspace 里的 HEARTBEAT.md，判断这一轮有没有事可做，没有就快速结束。不少刚上手的人会觉得"这不如事件推送优雅"，但把真实任务跑起来后会发现，轮询和推送各有适用面——问题不是二选一，而是怎么分层。

## 问题

两类机制的真实代价常被低估：

- **轮询的代价是持续消耗。** 每次 heartbeat 都是一次完整的模型调用，哪怕 agent 最后只回一句"无事可做"，token 照付。间隔越短，账单越陡。
- **推送的代价是工程兜底。** 每个事件源都要接 webhook 或写监听器，丢事件、重试、乱序、认证都得自己处理；而且很多外部服务根本不提供 webhook，想推也推不了。

## 做法

我目前的落地方式是按事件来源分层：

1. **先分类。** 把需求拆成"外部事件驱动"（CI 失败、邮件到达、PR 更新）和"内部状态巡检"（磁盘检查、日历汇总、定时摘要）两类。
2. **外部事件优先走推送。** 源服务有 webhook 的，通过插件或 MCP 转成一条消息直接投给 agent，按需触发，不烧空转 token。
3. **内部巡检交给 heartbeat。** HEARTBEAT.md 里只写可判定的检查项，让 agent 能干脆地输出"无事可做"。
4. **高频任务用 cron，不动全局间隔。** 某项检查需要每 5 分钟跑一次，就给这个任务单开 cron 表达式，而不是把 heartbeat 全局调密，殃及所有低频任务。
5. **配置活跃时段。** 夜间静默，避免半夜的无意义调用和对用户的打扰。

## 踩坑点

- **空 heartbeat 也烧钱。** 我曾把间隔调到 5 分钟试运行，一天下来光 heartbeat 就消耗几十万 token，其中九成是"无事可做"。先量化单次成本，再定频率。
- **HEARTBEAT.md 写得太开放。** 类似"看看有没有值得关注的"这种条目，agent 每轮都会认真思考并输出一大段。检查项要能明确回答是/否。
- **推送不是银弹。** webhook 服务方抖动一次，事件就丢了。关键事件要留一条低频轮询做对账补偿。
- **heartbeat 和 cron 重复调度。** 同一件事两边都配了，任务跑两遍、通知发两份。加新任务前先盘点已有调度。
- **会话膨胀。** heartbeat 全部进同一个 session 时，上下文越滚越大，成本和响应速度都会恶化，注意定期收敛或拆分会话。

## 可复用建议

- 默认采用"**低频 heartbeat 巡检 + 关键事件 webhook 推送**"的混合模式，最省心。
- 频率判断标准：滞后 15 分钟以上可接受 → heartbeat；要求分钟级实时 → cron 或推送。
- 把"无事可做"当成 heartbeat 的一等输出。HEARTBEAT.md 越克制，轮询越便宜。

## 总结

轮询买到的是简单和兜底，推送买到的是实时和精确，没有谁全面胜出。OpenClaw 的 heartbeat 适合承担低频巡检和兜底对账的角色；对实时性有硬要求的源，老老实实接 webhook。别为了架构洁癖把所有事都塞进 heartbeat，也别因为推送"看起来高级"就为每个不提供 webhook 的服务硬造轮子。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/2361e8b2a4e5157a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/80096cf76a4679ab.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/e4a3ca5a126f510d.png)

