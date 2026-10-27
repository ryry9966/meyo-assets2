---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 42781
source: 综合讨论
publishedAt: 2026-10-28
---

## 背景

OpenClaw 里有两套"让 agent 按时间做事"的机制，混用是新手最常见的问题：

- **cron**：真正的调度器。用 `openclaw cron add` 按 cron 表达式注册任务，到点由 gateway 触发，把一段固定的 prompt 塞进指定 session（隔离或主会话），跑完可投递到频道。
- **heartbeat**：固定间隔的心跳。agent 周期性被唤醒（默认约 30 分钟，可配置），读一遍工作区的 `HEARTBEAT.md`，自己判断"现在有没有该做的事"，没事就安静返回。

一个偏"到点执行"，一个偏"到点自查"。

## 问题

两类典型误用：

1. 把"每天 9 点发晨报"写进 HEARTBEAT.md，指望心跳自己算时间——结果是触发漂移、有时漏、有时重复。
2. 把"盯日历/收件箱，有变化就提醒我"建成每天跑一次的 cron——任务本身没有明确时刻，cron 只能瞎选时间点，覆盖率差还白烧 token。

本质区别：**cron 解决"什么时候做"，heartbeat 解决"要不要做"**。

## 做法

**heartbeat 侧：**

1. 把间隔调到符合容忍度的值（比如 1h 而不是默认 30m），这是心跳最主要的成本旋钮。
2. `HEARTBEAT.md` 当廉价检查清单写：每条都是"看一眼某个状态，有异常再说"，控制在几行以内。别放要跑长任务、翻多个数据源的活。

**cron 侧：**

1. 明确时刻的任务全部迁过去，例如：
   `openclaw cron add --name morning-brief --cron "30 8 * * *" --tz Asia/Shanghai --session isolated --message "汇总今日日历与未读，生成晨报"`
2. 需要对话上下文的用主会话；想保持干净的用 `--session isolated`，把所需状态显式写进 message。
3. 用 `openclaw cron list` 和 `openclaw cron runs` 复核触发记录，确认时区和补跑行为符合预期（参数细节以 `openclaw cron --help` 为准）。

**组合模式**：cron 负责定时抓数据、落盘或写状态；heartbeat 的清单只判断"数据里有没有值得说的"。定时产出 + 条件播报，各干各的。

## 踩坑点

- **心跳按次数计费**：间隔越短、清单越长，token 烧得越凶。哪怕只是返回"没事"，也是一次模型调用。
- **时区**：cron 表达式默认跟 gateway 所在机器走。服务器在 UTC、人在东八区，8 点的晨报会变成下午 4 点，务必显式指定时区。
- **隔离 session 无上下文**：message 里引用"刚才聊的"会直接失效，需要的信息都写全。
- **心跳告警有去重**：同一告警文本不会反复推送。别把"没收到消息"当成"没触发"，去日志确认。
- **心跳里放重活会阻塞**：一次心跳的长任务会占住主会话，期间正常对话排队等待。

## 可复用建议

一条决策规则就够用：

- 有明确触发**时间** → cron；
- 有明确触发**条件**但时间不定 → heartbeat（清单化，保持廉价）；
- 两者都要 → cron 定时产出状态，heartbeat 条件判断播报。

另外，定时任务上线后先观察几天 `cron runs` 和日志，确认语义正确，再缩短心跳间隔、增加清单条目。先少花钱验证，再放开频率。

## 总结

cron 和 heartbeat 不是二选一，而是分工：cron 是闹钟，heartbeat 是值班员。把"到点必须发生"的事交给 cron，把"发生了才知道该不该说"的事交给 heartbeat，成本和可靠性都会明显改善。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/cfa0012c2adf0983.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/cdf3195b41b3da47.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/892773c5f07f233d.png)

