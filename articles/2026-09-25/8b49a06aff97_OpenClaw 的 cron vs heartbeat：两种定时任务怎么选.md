---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 38914
source: 综合讨论
publishedAt: 2026-09-25
---

# 背景

在 OpenClaw 里做自动化，定时类需求有两条常用通道：**cron**（由 cron 工具管理的计划任务）和 **heartbeat**（心跳：Agent 按固定间隔醒来，读工作区的 `HEARTBEAT.md` 自查）。新手常见的两种跑偏：要么把所有周期任务都塞给 heartbeat，token 烧得莫名其妙；要么全押 cron，结果"该不该提醒"这类判断需求写出一堆脆弱的 if。这篇按实际使用经验，把两者的分工讲清楚。

# 两者差在哪

一句话：**cron 是时间驱动，heartbeat 是状态驱动的轮询加判断**。

- cron：写好表达式（或固定间隔），网关到点投递 payload 给 Agent。到点必跑，历史可用 `cron_runs` 查。它不思考"该不该跑"。
- heartbeat：Agent 默认每 30 分钟醒来一次，读 `HEARTBEAT.md`，由模型判断这次有没有值得说的事，没有就回 `HEARTBEAT_OK` 短路返回。每跳都花 token，换来的是"有情况才出声"。

# 做法

1. **先分类**。固定时间触发（日报、定时摘要、周末清理）走 cron；盯一个状态、有变化才提醒（监控目录、接口、收件箱异常）走 heartbeat。
2. **配 cron 定两件事**：会话目标和唤醒方式。重活（长文生成、批量抓取）用隔离会话，别污染主会话上下文；普通提醒投主会话即可。唤醒默认即时送达，也可选挂到下一次心跳。
3. **配 heartbeat 守三条**：`HEARTBEAT.md` 控制在十几行以内；明确写"无事返回 HEARTBEAT_OK，不要寒暄"；用 `activeHours` 把心跳限制在工作时段，砍掉半夜空转。
4. **混合用法**：heartbeat 做廉价巡检，发现异常由 Agent 立即上报，或临时补一条一次性 cron 兜底。

# 踩坑点

- **把 heartbeat 当调度器**：在 `HEARTBEAT.md` 里写"超过 9 点就发日报"，时间精度取决于心跳间隔，还可能重复发。时间触发请老实用 cron。
- **心跳间隔压太低**：每跳都是一次模型调用，输出只有 `HEARTBEAT_OK`，输入 token 照付。默认 30 分钟对多数场景够用，别为了"实时"压到几分钟。
- **cron 时区**：表达式按网关所在机器的本地时间算，跨时区部署先 `date` 确认再写，夏令时也要留心。
- **停机不补跑**：网关关着时 cron 和 heartbeat 都会漏，强一致需求要么外部触发，要么启动后跑一次补偿任务。
- **投递失败偏静默**：会话或渠道掉线时 cron 投递可能悄悄失败，定期查 `cron_runs`，别凭感觉认为"在跑"。
- **配置生效时机**：`HEARTBEAT.md` 每次心跳都会重读，改完下个周期生效；心跳间隔这类网关配置改完要重启才稳。

# 可复用建议

- 口诀：**时间驱动用 cron，状态驱动用 heartbeat**。判断标准是"到点就该做"，还是"值得才该说"。
- 主会话保持干净：重任务一律隔离会话，主会话只留对话和轻提醒。
- 一个任务只属于一条通道：cron 管日程，heartbeat 管筛选，不要重叠。
- 每周扫一眼 `cron_runs` 和 token 用量，心跳空转是费用膨胀的头号来源。

# 总结

cron 和 heartbeat 不是二选一，而是分层：cron 负责"什么时候做"，heartbeat 负责"要不要说"。各归其位之后，cron 的确定性和 Agent 的判断力都能用上，成本也可控。先分类任务，再选通道，上线后盯一周日志，上面这些坑基本都能绕开。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/0b4290f880a101f9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/13d51159fe2a911d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/cfa361c5c09c0689.png)

