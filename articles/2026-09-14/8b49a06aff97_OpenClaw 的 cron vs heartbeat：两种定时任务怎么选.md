---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 37438
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

OpenClaw 里能让 agent"定时干活"的路径有两条：

- **cron**：传统 cron 表达式触发，到点执行一段你写好的 prompt，产物可以投递到指定渠道，有完整的任务清单和执行记录。
- **heartbeat**：固定间隔的轮询，每次心跳会触发 agent 读一遍 workspace 里的 `HEARTBEAT.md`，由 agent 自己判断"有没有事要做"。没事就回 `HEARTBEAT_OK`，不打扰你。

很多同学两条都用过，但经常选反了，结果要么 token 烧得心疼，要么任务静默失败。

## 问题

典型误用有两种：

1. 把 heartbeat 当 cron 用——"每 5 分钟心跳一次帮我推送日报"。间隔短、耗时久，一天几百次模型调用，而推送时间还不精确（心跳是间隔制，不是定点制）。
2. 把该用 heartbeat 的场景硬塞进 cron——在 prompt 里写"先检查 A，如果 A 没变就……如果变了再……"。本质上你是在用 prompt 模拟事件监听，脆弱且难维护。

两者的心智模型不同：**cron 是"时间到 → 执行已写好的动作"，heartbeat 是"到点看一眼 → agent 决定要不要做事"**。

## 做法

**第一步：给任务分类。** 能写成"每周一 9:00 拉取数据并汇总推送"的 → cron；只能写成"帮我盯着 XX，有变化就告诉我"的 → heartbeat。

**第二步：cron 的正确用法。** 用 `openclaw cron add` 定义任务（名称、表达式、prompt、投递方式，具体参数以 `openclaw cron add --help` 为准），`cron list` 看清单，`cron runs` 查历史。注意 prompt 要**自包含**：cron 触发的是一次全新的 agent 会话，别指望它记得上次聊了什么。

**第三步：heartbeat 的正确用法。** 在 workspace 写一份 `HEARTBEAT.md`，内容是短清单（建议 10 行内），把"什么算有事"的判断标准写清楚；用 `every` 控制间隔、`activeHours` 限定活跃时段；无事可做时明确让 agent 返回 `HEARTBEAT_OK`，不产出消息。

## 踩坑点

- **心跳成本**：每次心跳都是一次真实的模型调用，即使最终只回 `HEARTBEAT_OK` 也计 token。间隔设 5 分钟，账单会很诚实。
- **深夜静默**：heartbeat 默认只在 active hours 内运行，如果你依赖它夜间触发，会"安静地不执行"。夜间任务请交给 cron。
- **时区**：cron 表达式按 gateway 配置的时区解释，服务器在 UTC 时，"每天 9 点"可能是北京时间 17 点。显式指定时区，别赌默认值。
- **重复提醒**：同一件事既配了 cron 又写进 `HEARTBEAT.md`，会收到两份通知。一个职责只给一个机制。
- **清单太长**：整份 `HEARTBEAT.md` 会作为上下文进入每一次心跳，写 50 行等于每次心跳多付 50 行的 token。
- **心跳不补跑**：gateway 离线期间错过的心跳不会事后补执行。对"一次都不能漏"的任务，用 cron 并定期看执行记录。

## 可复用建议

- 一句话判据：**时间确定用 cron，条件确定用 heartbeat**。
- `HEARTBEAT.md` 控制在十几行，只放"值得唤醒 agent"的检查项，其余交给 cron 或手动触发。
- 给每个 heartbeat 配一个日志出口（比如追加到一个状态文件或专用频道），方便复盘它到底实际触发了几次。
- cron 的 prompt 里写清楚输出格式和投递目标，把它当 API 用，而不是当聊天用。

## 总结

cron 和 heartbeat 不是竞争关系，而是"精确闹钟"与"值班巡检"的分工。把确定性动作交给 cron，把模糊监控交给 heartbeat，同时严格控制心跳频率和清单长度——OpenClaw 的自动化才能既省 token，又不打扰人。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/4af39986590a5b0f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/3bda64214dfdd961.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/5563ab14773f3115.png)

