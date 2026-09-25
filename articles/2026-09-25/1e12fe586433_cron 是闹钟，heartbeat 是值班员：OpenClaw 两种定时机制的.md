---
title: cron 是闹钟，heartbeat 是值班员：OpenClaw 两种定时机制的选择
feedId: 38918
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

OpenClaw 里有两套"让 agent 定时干活"的机制：**cron 任务**和 **heartbeat（心跳）**。新手上手时经常把两者混着用，结果要么 token 烧得莫名其妙，要么任务在错误的时间没跑、或者跑得不该跑。这篇把两者的定位差异和选择方法梳理清楚。

## 问题：两者到底差在哪

- **cron**：时间驱动。到点执行一次给定的 prompt，分钟级精度，跑完即止。
- **heartbeat**：注意力驱动。gateway 按固定间隔（默认 30 分钟）唤醒 agent，agent 读一遍工作区的 HEARTBEAT.md，自己判断"有没有事需要出声"，没事就保持沉默。

三个关键差异：

1. **精度与成本**：cron 只在触发时产生一次调用；heartbeat 每个间隔都消耗一次，频率越高成本越线性上涨。
2. **语义**：cron 是"执行这个动作"，heartbeat 是"评估这个状态"。
3. **上下文**：cron 可以选隔离 session 或投递到主会话；heartbeat 有自己独立的 session，与主对话完全隔离。

## 做法

先判断任务性质：

- 固定时刻、确定性动作（早报、定时摘要、定时抓取）→ **cron**
- "盯一下、有事再说话"的环境监控（盯文件变动、盯服务状态、盯收件箱）→ **heartbeat**

**cron 配置步骤：**

1. 用 `openclaw cron add` 或直接编辑 `~/.openclaw/cron/jobs.json` 定义任务；
2. prompt 写成自包含指令，不假设任何上下文；
3. 显式指定时区，别赌服务器默认时区；
4. 用 `openclaw cron list` 加一次手动触发，先验证一轮再上线。

**heartbeat 配置步骤：**

1. 在配置里把 `agents.defaults.heartbeat.every` 调到 30m 起步；
2. 维护 `~/.openclaw/workspace/HEARTBEAT.md`：一行一个检查项，带明确阈值（"磁盘使用 > 90% 才报"）；
3. 写清"沉默优先"规则：什么情况必须出声、什么情况直接忽略；
4. 观察几天 session 日志，把误报项逐条收紧。

## 踩坑点

- **间隔调太短**：有人把 heartbeat 调到 5 分钟，token 成本翻了几倍，而且大多数节拍什么都没做——从 30m 起步，确实不够再加。
- **HEARTBEAT.md 写成流水账**：每次唤醒模型都要读一大篇，成本高，判断质量反而下降。
- **cron prompt 引用对话历史**：写了"继续我们刚才讨论的内容"，但任务用的是隔离 session，什么都拿不到。
- **时区问题**：服务器是 UTC，cron 没显式指定时区，早报在下午才发。
- **把精确间隔任务交给 heartbeat**：实际间隔 = 配置间隔 + 执行时长，天然有抖动，要精确就老实用 cron。
- **补跑语义不同**：gateway 停机后两者的补跑/跳过行为不一样，依赖"停机期间也不漏"的任务，请针对你当前版本自己验证，别想当然。

## 可复用建议

- 一句口诀：**时间确定性找 cron，注意力管理找 heartbeat**。
- HEARTBEAT.md 当作值班清单维护：一条一行、带阈值，定期删过时项。
- cron 任务尽量幂等：同一天补跑两次也不会出问题。
- 组合用法很实用：heartbeat 负责盯异常，cron 负责每天固定时间出汇总，各干各的，互不抢活。

## 总结

cron 是闹钟，heartbeat 是值班员。闹钟负责准点响，值班员负责"没事别吵我"。按任务的驱动类型选机制，成本、可靠性和打扰程度都会好很多。拿不准的时候，先用 cron 跑起来，再考虑要不要降级成 heartbeat。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/d2cf235ea8ff267a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/42dcb8b4566841de.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/40162b835343026d.png)

