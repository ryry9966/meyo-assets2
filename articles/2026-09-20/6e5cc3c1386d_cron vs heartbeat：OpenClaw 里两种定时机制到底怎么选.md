---
title: cron vs heartbeat：OpenClaw 里两种定时机制到底怎么选
feedId: 38211
source: 综合讨论
publishedAt: 2026-09-20
---

## 背景

OpenClaw 的常驻网关给了 Agent 两套"动起来"的机制：**cron**（按表达式或间隔精确触发一个 prompt）和 **heartbeat**（默认每 30 分钟向主会话发一次 tick，Agent 自己读工作区的 HEARTBEAT.md 决定要不要干活）。

新手常见做法是把所有周期性需求都塞进其中一种，结果要么时间不准，要么 token 烧得莫名其妙。

## 问题在哪

两类需求本质不同：

- "每天 8:30 汇总昨日日志发到频道"——时刻敏感、动作确定；
- "定期看看收件箱有没有值得提醒我的东西"——时刻不敏感，要不要干活由 Agent 判断。

拿 cron 做条件检查，所有判断逻辑得写死在 prompt 里；拿 heartbeat 做定时任务，30 分钟的间隔抖动会让"8 点的报告"变成"8 点到 8 点半之间某刻的报告"。

## 怎么做

1. **先分类**：列出所有周期性需求，按"固定时刻"和"条件触发"分成两列。
2. **固定时刻 → cron**：`openclaw cron add` 用 5 段表达式或 interval 定义；建议用 isolated session，避免污染主会话上下文；想让结果推送到 IM，把 delivery 设为 announce 并指定 channel。也可以直接在对话里说"每天 8:30 帮我……"，让 Agent 自己建 job。
3. **条件触发 → heartbeat**：检查项写进 HEARTBEAT.md，控制在 3–5 条；明确写上"无事可做时回复 HEARTBEAT_OK 不输出"，这是省 token 的关键。
4. **验证**：各自手动触发一次，跑一周后回看日志和 token 账单再调参。

## 踩坑点

- **HEARTBEAT.md 写成大而全的待办清单**：每个 tick 都触发完整推理，一天 48 次起步，账单很惊人。检查项越少越具体越好。
- **cron 走 main session**：定时输出混进日常对话上下文，会话越滚越大，回复质量明显下滑。
- **时区**：cron 按网关本地时间解释，服务器在 UTC 而你在 UTC+8，报告会晚 8 小时。部署时把 TZ 显式设好。
- **错过的 tick 不补跑**：网关重启期间到点的 job 就丢了。重要任务要么接受丢失，要么在任务内自查"上次执行时间"。
- **别指望 heartbeat 做秒级响应**：你正在聊天时 tick 可能排队或被抑制。

## 可复用建议

- 判断标准一句话：**说得出确切触发时刻的用 cron，只能说"定期看一眼"的用 heartbeat。**
- 混合模式更实用：cron 负责准时开跑，任务 prompt 第一步先检查数据源，没新东西直接结束——把 heartbeat 的"判断"内联进 cron，确定性和隔离性都更好。
- 定期审计：`openclaw cron list` 过一遍删掉失效 job，HEARTBEAT.md 每月修剪一次。

## 总结

cron 是闹钟，heartbeat 是巡逻。闹钟解决"什么时候必须做"，巡逻解决"要不要做"。任务分类清楚、HEARTBEAT.md 保持克制、cron 一律隔离会话——这套体系就能长期稳定跑下去，token 账单也不会变成玄学。

---

