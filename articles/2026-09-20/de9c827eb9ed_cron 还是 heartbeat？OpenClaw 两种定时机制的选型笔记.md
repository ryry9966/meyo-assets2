---
title: cron 还是 heartbeat？OpenClaw 两种定时机制的选型笔记
feedId: 38230
source: 综合讨论
publishedAt: 2026-09-20
---

## 背景

OpenClaw 的常驻 agent 有两条“时间线”：一条是 `cron` 工具——你给它 cron 表达式和一段任务描述，到点它拉起一次执行；另一条是 heartbeat——主会话按固定间隔被唤醒一次，读一遍工作区里的 HEARTBEAT.md，由模型自行判断这一轮要不要干活。刚上手时很容易把两者混着用，我在自己的实例上跑了一个多月，把选型经验整理成这篇。

## 问题

混用的典型症状有三个：

1. 需要 9:00 准点发出的日报，写进了 HEARTBEAT.md，结果 9:40 才到，有时干脆被跳过；
2. 心跳清单越堆越长，token 消耗和 agent 的“自由发挥”一起上涨，半夜被主动汇报吵醒；
3. cron 任务到点跑了，但结果没投递到任何 channel，隔了三天才发现。

## 两者的本质差异

- **触发语义**：cron 是“承诺”——表达式命中就必须执行；heartbeat 是“巡视”——每 30 分钟（可配）看一眼，有没有值得做的事由模型判断。
- **上下文**：cron 默认在隔离会话里跑，干净、不污染主对话（需要上下文时可指定 `session: main`）；heartbeat 天然携带主会话全部上下文，适合“记得前因后果”的判断。
- **成本**：cron 只在触发时产生一次调用；heartbeat 每次心跳都是一次主会话调用（较新版本在清单为空时会直接跳过，不再烧 token）。

## 做法

**cron 侧：**

1. `cron add`：起个能认出来的名字，写 5 位表达式和任务 message；
2. 显式配置结果投递（deliver 到哪个 channel），这是最容易漏的一步；
3. `cron list` 确认后，手动 `cron run` 触发一次验证，之后用 `cron runs <jobId>` 看执行历史；
4. 一次性提醒用完记得 `cron remove`，别让任务列表变垃圾场。

**heartbeat 侧：**

1. 在工作区建 HEARTBEAT.md，写成 checklist，上限 3–5 条；
2. 每条必须包含触发条件和“不满足就跳过”的说明，例如：“仅在 ~/reports 出现新文件时做摘要并通知，否则跳过”；
3. 间隔从默认值起步，别贪快；如果版本支持 activeHours，把夜间时段关掉；
4. 观察一周，逐条做减法。

**组合用法**：时间敏感、有明确时刻的事全部走 cron；“持续盯着某处、需要对话上下文”的巡检留给 heartbeat。重活让 cron 起隔离会话，避免长任务堵住主会话、心跳跟着排队。

## 踩坑点

- **时区**：cron 按宿主机时区解析，容器里默认 UTC，“早八”会变“下午四点”。部署后先核对系统时间。
- **心跳不保证准点**：agent 忙或会话锁定时心跳会漂移甚至跳过，任何“每天 X 点必须……”的需求都不要写进 HEARTBEAT.md。
- **HEARTBEAT.md 写成愿望清单**是最大的坑：模型会主动找活干。每条都要能回答“什么情况才算触发”。
- **主会话上下文很大时**，每次心跳都不便宜，心跳频率是最容易被低估的成本项。
- **cron 忘配投递**，结果留在隔离会话里无人认领；排查第一入口是 `cron runs`。

## 可复用建议

- 一句话选型：**到点的交给 cron，“盯着”的交给 heartbeat**；两者都要时，cron 触发并投递到主会话所在的 channel。
- 把 HEARTBEAT.md 当作“最佳努力”清单，而不是承诺清单。
- cron 任务统一命名（如 daily-report、weekly-digest），每月 `cron list` 清理一次。
- 心跳新增条目先注释试运行三天，确认不误触再正式启用。
- 排障顺序固定：cron 看 runs 历史，heartbeat 看主会话日志里每轮唤醒的决策记录。

## 总结

cron 和 heartbeat 不是竞争关系，而是两种执行语义：前者是确定性的时刻承诺，后者是不确定性的周期巡视。把“必须几点做”和“有空看一眼”分开之后，token 账单、准点率和半夜打扰这三个问题基本都能压下去。我的实例现在 heartbeat 只剩两条真正的巡检，其余十来个任务全在 cron 里，跑了一个月没再出过幺蛾子。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/825a143f18843807.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/f641753714d514f9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/e6133c5d6b6834fa.png)

