---
title: HEARTBEAT.md 实践：让 Agent 主动值班，而不是等你提问
feedId: 40110
source: 综合讨论
publishedAt: 2026-10-02
---

# 背景

大多数人用 Agent 的方式仍是"一问一答"：你不开口，它就永远停着。OpenClaw 有个朴素但容易被忽略的机制——心跳（heartbeat）：Gateway 按固定间隔（默认 30 分钟，可通过 `HEARTBEAT_INTERVAL_MS` 调整）唤醒一次 Agent，让它读取 workspace 下的 `HEARTBEAT.md`，自己判断有没有该做的事。用对了，Agent 就从问答机器变成值班助理。

# 问题

我最初把 HEARTBEAT.md 当许愿池，塞了十几条"帮我盯着 XX"。结果两天后弃用：token 烧得快，每天收到七八条"一切正常"的噪音，还有一条半夜三点把我吵醒。根源是我把"愿望清单"当成了"值班手册"——心跳需要的是明确的触发条件、动作和沉默规则，而不是模糊的意图。

# 做法

1. 打开 `~/.openclaw/workspace/HEARTBEAT.md`。默认只有一行占位标题，**空文件或纯占位符会直接跳过本次心跳**——不调模型、不花 token。这是设计上的好习惯：没任务就让它安静。
2. 把任务写成 checklist，每条一句话，包含三要素：触发条件 + 动作 + 沉默规则。示例：

```markdown
# HEARTBEAT

- 心跳时检查主服务器磁盘使用率，超过 85% 才发消息并附 df 输出
- 每天上午第一次心跳：汇总最近 24 小时 git 提交，无提交则不发言
- ~/inbox 出现新 PDF 时归档并简述内容
- 以上都不满足：回复 HEARTBEAT_OK
```

3. 调整节奏：interval 按需缩短或拉长，同时配 activeHours（如 08:00–22:00）避免夜间打扰。不同版本字段名略有差异，以当时的 docs 为准。
4. 观察一周心跳日志，删掉从未触发的条目和天天误报的条目，逐步收敛写法。

# 踩坑点

- **写了内容就计费**。只要有有效任务，每次心跳都会跑一次模型。条目控制在 3–5 条，措辞要短。
- **没有沉默条件就是自我轰炸**。"帮我看看服务器"这种写法会让它每次心跳都汇报。必须写清楚"什么情况下才出声"。
- **无事时让它回 `HEARTBEAT_OK`**。这是约定的静默回复，网关会吞掉不投递；重复内容也有去重，但别依赖。
- **心跳跑在主会话里**，内容会进上下文。别把它当 cron 用，更别塞长任务。
- **重活别放心跳**。跑 10 分钟的脚本会拖垮整个节奏。心跳只做轻量"检查 + 判断"，执行交给脚本或 MCP 工具，需要人拍板时再说话。
- **时区**。容器里 gateway 的时区和本地不一致，activeHours 会整体错位，先 `date` 验证。

# 可复用建议

- 心跳与 cron 分工：固定时间点的事（每天 9 点日报）用 cron；"满足条件就做"的事（磁盘超阈值）用心跳。
- 把 HEARTBEAT.md 当值班手册维护：每周 review 一次，一条任务若一个月没触发或天天触发，改写或删掉。
- 让心跳输出尽量"变更驱动"：只报告 diff，不报告全量。

# 总结

HEARTBEAT.md 的价值不在"能定时跑"，而在逼你把模糊的期待写成可判定的规则。三五行 checklist、明确的沉默条件、配合 activeHours 和 cron 分工，Agent 就能从被动应答变成真正的值班助理——安静、便宜、该出声时才出声。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/1c5f9b5d8b8508af.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/495c9cbd93b7b41d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/572eb61fe89b870d.png)

