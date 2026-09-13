---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 37426
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

OpenClaw 里有两种"定时"能力，边界经常被搞混：

- **cron**：由 gateway 内置调度器驱动，按 cron 表达式或固定间隔触发一次任务投递，可以打到主会话，也可以开隔离会话；
- **heartbeat**：周期性心跳（默认约 30 分钟一次），把 `HEARTBEAT.md` 的内容送进主会话，由 agent 自己判断这一跳要不要干活。

两者都能实现"定时做事"，但执行模型完全不同，选错机制是社区里最常见的返工原因。

## 问题

典型错误用法有两类：

1. **把所有周期任务都塞进 heartbeat**。结果 agent 每半小时在主会话里"认真回答"一遍不需要回答的问题，token 账单很难看。
2. **用 cron 做条件式巡检**。到点必须投递，哪怕无事发生也跑一轮完整推理，又贵又不优雅。

本质区别一句话：**cron 是时间驱动、确定触发；heartbeat 是周期唤醒、条件执行。**

## 做法与步骤

**第一步：给任务分类。**

- 固定时间点、必须准时：日报、定时提醒、收盘后汇总 → cron
- "每隔一阵看一眼，没事就闭嘴"：盯仓库更新、查设备状态、扫收件箱 → heartbeat

**第二步：确定性任务交给 cron。**

```bash
openclaw cron add --name daily-digest \
  --cron "0 9 * * *" \
  --session isolated \
  --message "汇总昨天的会话要点，输出简报"
```

隔离会话不占主上下文，产物再通过通知/投递送回主会话。

**第三步：条件巡检交给 heartbeat。** 把检查项写进 `HEARTBEAT.md`，并明确"无操作"规则：

```markdown
## 检查项
- 有未处理的 CI 失败吗？有就修复并通知我
- 关注的仓库有新 release 吗？有就总结变更
## 规则
- 全部无事：直接跳过，不要回复
```

心跳间隔在配置里调（`agents.defaults.heartbeat.every`），先用短周期观察，再放大。

**第四步：混合编排。** 实际项目里两者是配合关系：heartbeat 做轻量巡检，发现异常时 agent 立即行动；cron 负责每天到点必发的汇总，互不抢活。

## 踩坑点

- **heartbeat 烧 token 的根因**是 HEARTBEAT.md 里没写"无事跳过"，agent 默认会对每一跳礼貌回复。
- **cron 隔离会话没有记忆**。需要的上下文（项目路径、上次的状态）必须写进 prompt，别指望它"记得上次"。
- **时区**：cron 表达式按 gateway 所在机器的时区跑。服务器是 UTC、人在东八区，日报会早到 8 小时。
- **heartbeat 不保证准点**，间隔只是约数，还有队列延迟；别拿它做"9:00 必须发出"的事。
- **HEARTBEAT.md 越写越长**，执行会走样。控制在个位数条目，复杂逻辑拆给 cron 或子 agent。

## 可复用建议

- 决策口诀：**到点必做用 cron，见机行事用 heartbeat；"没事别吵我"是 heartbeat 的使用前提。**
- 成本心算：heartbeat 成本 ≈ 触发频率 × 主会话上下文大小；cron（隔离会话）成本 ≈ 任务 prompt 本身。频率高、上下文大时，优先 cron 隔离会话。
- 同一件事只配置一处。避免 cron 和 heartbeat 各写一套相似逻辑，之后改需求只改了一半。
- 上线前先跑一两天，用 `openclaw cron list` / 运行记录和会话日志确认触发频率与实际耗时，再固化周期。

## 总结

cron 和 heartbeat 不是替代关系，而是分工关系：cron 管"什么时候必须做"，heartbeat 管"周期性地看有没有该做的"。按**时间确定性**和**是否依赖主会话上下文**两个维度给任务归类，大多数选型困难会自然消失。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/65a13a07e6677137.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/06dc76f654cc3151.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/c1df98ff3cd954a5.png)

