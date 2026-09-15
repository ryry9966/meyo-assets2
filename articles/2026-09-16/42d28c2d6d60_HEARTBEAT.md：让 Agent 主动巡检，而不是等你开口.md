---
title: HEARTBEAT.md：让 Agent 主动巡检，而不是等你开口
feedId: 37731
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

OpenClaw 的交互默认是被动式的：你发消息，它回消息。但 gateway 内置了一个经常被忽略的机制——心跳。每隔一段时间（默认 30 分钟，可配），gateway 会用一条极简的提示词唤醒 agent，agent 会去读工作区里的 `HEARTBEAT.md`，按里面的约定决定要不要做事、要不要汇报。用好这一个文件，等于给 agent 装上了定时自检器。

## 问题

实际用下来常见两个极端。一是文件空着或只有模板注释，心跳空转：每 30 分钟烧一次 token，零产出。二是往里面塞十几条任务，结果 agent 每个周期都在群里刷屏"我检查了 X，一切正常"。本质都是没把这份文件当成一份**运行规约**来写，只当成了待办清单。

## 做法

1. **定位文件**：工作区根目录，默认 `~/.openclaw/workspace/HEARTBEAT.md`。
2. **三段式写任务**：触发条件、动作、汇报规则，缺一不可：

```markdown
# HEARTBEAT.md

## 任务1：错误日志巡检
- 触发：每次心跳
- 动作：检查 ~/logs/app.err 自上次运行后的新增行
- 汇报：有新增 → 摘要前 20 行发到 #ops；无 → 回复 HEARTBEAT_OK
- 状态：把上次读取的文件位置写入 .state/heartbeat.json
```

3. **配置 gateway**：`interval` 调试期可降到 5 分钟观察行为，稳定后调回；`target` 指定主动消息发往哪个频道，不配就会落到默认会话甚至丢失；`activeHours` 限制在工作时段跑，避免半夜折腾。
4. **约定沉默协议**：没有值得说的事就让 agent 回复 `HEARTBEAT_OK`，gateway 不会把这条转发到频道——这是防刷屏的关键开关。
5. **状态闭环**：强制 agent 把"上次运行时间 + 结果"写回状态文件，否则同一任务会被反复执行。

## 踩坑点

- **Token 成本**：心跳是固定开销，文件里别放大段上下文，更别让它"顺便浏览网页"。重活交给 cron 或 skills，心跳只做轻量判断和路由。
- **任务没有终态**：写成"持续关注 X"的任务，agent 每个周期都会从头再做一遍。一定要有可写的状态。
- **时区问题**：`activeHours` 按 gateway 本地时间计算，Docker 容器里默认 UTC 是高频事故来源，先 `date` 确认再上配置。
- **静默失败**：心跳出错默认只进 gateway 日志，行为异常时先查日志，别靠猜。

## 可复用建议

把 HEARTBEAT.md 当**路由表**而不是任务清单：每条任务只回答"什么条件、做什么、何时闭嘴"。同时活跃任务控制在 3 条以内，每周修剪一次，删掉一周内零汇报的任务——它们要么无价值，要么条件写错了。重要告警和例行汇报分流到不同频道，避免噪音淹没信号。

## 总结

HEARTBEAT.md 的价值不在于"自动化了多少事"，而在于把 agent 从应答器变成有主动巡检能力的常驻进程。写清触发、动作、沉默规则，配上 `target` 和 `activeHours`，半小时内就能跑起来。建议从一条最简单的日志巡检起步，比一次写十条任务靠谱得多——心跳机制是用来长期养习惯的，不是用来一次性堆需求的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/6af6805a2664d2a6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/af1e12eb4b15176b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/6b5e92be62a59fc8.png)

