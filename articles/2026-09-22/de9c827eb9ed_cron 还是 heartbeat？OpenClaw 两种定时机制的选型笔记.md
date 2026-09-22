---
title: cron 还是 heartbeat？OpenClaw 两种定时机制的选型笔记
feedId: 38518
source: 综合讨论
publishedAt: 2026-09-22
---

## 背景

OpenClaw 里有两套让 agent "按时间干活"的机制：**cron** 和 **heartbeat**。刚上手时很容易混着用，我最初就把所有事都塞进了 heartbeat，跑了两周才理清两者的边界。

先说本质区别：

- **cron**：时钟驱动。标准 cron 表达式，到点就触发一次你预先写好的任务，确定性很强。
- **heartbeat**：间隔驱动 + agent 自治。agent 每隔 N 分钟醒来一次，读取工作区的 `HEARTBEAT.md`，自己判断"这次有没有事要做"。没事就回复 `HEARTBEAT_OK` 保持静默。

## 问题：全塞进 heartbeat 的代价

我的第一版配置里，日报、整点提醒、仓库 issue 巡检全靠 heartbeat。两周后暴露两个问题：

1. **时间不准**。heartbeat 是"每 30 分钟一次"的间隔循环，不是时钟对齐的。"每天早上 8 点出日报"这种需求它天生做不到。
2. **成本失控**。heartbeat 静默不等于零成本，每一轮都是一次真实的模型调用。巡检项写得模糊时，agent 还会频繁"动手"，进一步放大消耗。

## 做法：按两个维度分类

我把任务清单按两个问题过筛：**是否依赖绝对时间？是否需要 agent 自行判断做不做？**

**绝对时间类 → cron。** 用 CLI 注册：

```bash
openclaw cron add --name daily-digest \
  --cron "0 8 * * *" --tz Asia/Shanghai \
  --message "汇总昨天的会话记录，生成日报发到群里"
openclaw cron list   # 核对注册结果
```

（版本迭代较快，具体 flag 以 `openclaw cron add --help` 为准。）

**巡检类 → heartbeat。** 在 workspace 的 `HEARTBEAT.md` 里写检查项，关键是给每一项明确的触发阈值，比如"未读消息超过 5 条才整理"，让 agent 能快速否决。间隔按成本水位调，不必拘泥默认值。

跑一周后用 `openclaw cron` 的运行记录和会话日志核对实际触发情况，再回调参数。

## 踩坑点

- **时区**。容器里默认 UTC，cron 不显式指定时区，"早八"会静默变成"下午四点"。这个坑我踩过一次，日报晚了 8 小时才发现。
- **"每 30 分钟"不等于"整点对齐"**。heartbeat 从进程启动时刻起算，重启后重新计时，别指望它对齐整点。
- **heartbeat 占用主会话**。它默认跑在主 session 里，巡检太频繁会稀释上下文，正在进行的正经对话容易被冲淡。巡检逻辑尽量收敛在 `HEARTBEAT.md` 里，不要给它自由发挥的空间。
- **cron 任务污染主会话**。跑长任务的 cron 建议用隔离会话，输出通过投递通道拿回来，主会话保持干净。
- **HEARTBEAT.md 写得太宽泛**。"帮我看看有没有重要的事"这种描述，等于把否决权交给模型的即时心情，无效动作会明显变多。

## 可复用建议

- 一句话选型：**cron 管日程，heartbeat 管嗅觉**。必须发生在确切时刻的事用 cron；时刻不敏感、需要 agent 判断"要不要做"的事用 heartbeat。
- `HEARTBEAT.md` 的每一项都写清"何时才值得动手"，可否决性是这个机制省钱的核心。
- cron 负责"做事"，不要在 cron 的 prompt 里塞判断逻辑；判断逻辑属于 heartbeat。
- 每月清理一次 `openclaw cron list`，过期任务留着只会产生幽灵消息。

## 总结

cron 是确定性的日程表，heartbeat 是低精度的巡逻兵。两者不是竞争关系，而是分层：时间层交给 cron，决策层交给 heartbeat。选型只看一个问题，**这件事是否必须发生在某个确切时刻？** 是，就 cron；否，且需要 agent 自己掂量，就 heartbeat。

把这条边界划清楚之后，我的定时任务成本降了一大半，触发准确性也回来了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/b8bc31dd6e977157.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/81519dd264163a2b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/06c14425393c7c63.png)

