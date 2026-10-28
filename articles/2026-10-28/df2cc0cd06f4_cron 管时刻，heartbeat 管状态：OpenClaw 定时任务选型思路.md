---
title: cron 管时刻，heartbeat 管状态：OpenClaw 定时任务选型思路
feedId: 42811
source: 综合讨论
publishedAt: 2026-10-28
---

## 背景

OpenClaw 提供了两种让 agent "定时动起来"的机制：cron job 和 heartbeat。不少新手的第一个疑问是：功能看着重叠，到底用哪个？这篇基于我们几个自动化项目的实践，给出一个可操作的判断框架。

## 两者的本质区别

一句话：**cron 是时间驱动，heartbeat 是状态驱动**。

- **cron**：按 cron 表达式在精确时刻触发，prompt 创建时就固定，每次运行有记录。语义是"几点几分做某事"。
- **heartbeat**：按固定间隔（默认约 30 分钟）醒来一次，读取 workspace 里的 HEARTBEAT.md，由模型判断本次干不干活、干多少。语义是"醒来看一眼，有事再动"。

由此推出四个实际差异：精确度（cron 分钟级，heartbeat 会漂移）；决策者（cron 是人，heartbeat 是模型）；成本模型（cron 不跑不花钱，heartbeat 每跳都有 token 开销）；上下文（cron 可用独立 session，heartbeat 默认走主会话）。

## 怎么配

### 固定时刻 → cron

```bash
openclaw cron add --name "weekly-digest" \
  --cron "0 9 * * 1" \
  --prompt "读取上周运行日志，输出三段式摘要发到通知群"
```

适合日报周报、定时备份、定点抓取。注意：独立 session 没有主会话上下文，任务需要的状态要么写进 prompt，要么让任务先读某个状态文件。

### 固定条件、灵活时刻 → heartbeat

两步：在 workspace 的 HEARTBEAT.md 写检查项，在配置里调间隔（具体键名以你版本文档为准）。条目建议按"条件 + 动作 + 静默规则"写：

```md
- 若 /tmp/deploy.lock 存在则提醒我，否则不要回复
- 磁盘使用率超过 85% 才提醒，同一问题 24 小时内只报一次
```

静默规则很重要：没有它，agent 每次心跳都可能在"确认没异常"上空耗，甚至反复打扰你。

## 踩坑点

1. **heartbeat 空转也烧钱**。HEARTBEAT.md 越长每跳越贵，控制在三五条以内，低频检查合并成"每天首次心跳做一次"。
2. **模糊条目等于没写**。"帮我盯着服务"这种条目，模型会自己脑补判断标准，容易漏报或误报。条件必须可验证：读某个文件、跑某条命令。
3. **重活别放 heartbeat**。心跳跑在主会话，一个跑几分钟的任务会搅乱你的日常对话。重任务交给 cron 的独立 session，或由心跳触发后转给子 agent。
4. **时区**。cron 表达式按 gateway 所在机器的时区解析，服务器在 UTC 时你写的"9 点"未必是你以为的 9 点，先 `date` 确认再建任务。
5. **双机制重叠**。cron 和 heartbeat 同时做同类检查会收到重复通知，同一件事只归属一个机制。
6. **两者都不适合事件级响应**。"文件一变就动"的场景用事件钩子或 MCP 工具触发，别靠心跳轮询硬凑。

## 可复用建议

我们的选型口诀：**固定时刻用 cron；固定条件、灵活时刻用 heartbeat；事件触发用 hook**。把 heartbeat 当巡检表，把 cron 当班表。

另外两条经验：调试期的 cron 任务先禁用而不是删除，`openclaw cron list` 留个底方便对比改动；任务命名和输出去向（日志文件、通知渠道）写清楚，一个月后你还认得出它是干嘛的。

## 总结

cron 和 heartbeat 不是竞争关系，而是两种触发语义：一个对时间负责，一个对状态负责。判断标准只有一个——这件事是必须在某个时刻做，还是满足条件就该做。前者给 cron，后者给 heartbeat，事件驱动的交给 hook。分清这三层，定时自动化基本不会再纠结。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/51daed6347b2684c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/a1f278fa0f72c78d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/9031f74197def39a.png)

