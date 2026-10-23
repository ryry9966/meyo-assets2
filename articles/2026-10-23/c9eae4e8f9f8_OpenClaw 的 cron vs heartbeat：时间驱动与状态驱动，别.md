---
title: OpenClaw 的 cron vs heartbeat：时间驱动与状态驱动，别再混用
feedId: 42378
source: 综合讨论
publishedAt: 2026-10-23
---

## 背景

OpenClaw 里有两条“定时”路径：一条是调度器里的 **cron 任务**，用 cron 表达式在指定时间触发；另一条是 **heartbeat**，按固定间隔（默认半小时左右）唤醒主会话，让 agent 自己读工作区的 `HEARTBEAT.md` 决定要不要做事。两者都能“定时”，但设计意图完全不同。

## 问题

我最初把所有周期性需求都塞进了 `HEARTBEAT.md`：日报、提醒、巡检全靠心跳。结果一是 token 消耗明显上涨——每次 beat 都要带着整份文件过一遍模型；二是时间敏感的任务不可靠，让 agent 在心跳里判断“现在是不是 8 点半”，实测会提前、延后甚至漏掉。反过来，也有人把需要会话上下文的巡检写成 cron，每次独立会话冷启动，上下文全靠 prompt 里重新交代。

## 做法

先给一个判断标准：

1. **需求能写成“在 T 时刻做 X”→ 用 cron。** 例如每天 08:30 推送日报、每周一汇总 issue、临时的一次性提醒。cron 任务是独立会话，prompt 自带完整指令，执行完把结果投递到指定 channel，与主会话互不干扰。
2. **需求只能写成“每隔一会儿看看有没有要做的”→ 用 heartbeat。** 例如“收件箱有未处理消息就提醒我”“某文件变了就整理归档”“待办是否有到期项”。它没有确定触发时刻，只有触发条件，条件判断交给 agent。
3. **组合用法：** cron 负责确定性时间点（兜底、日报），heartbeat 负责状态型巡检（条件触发、即时响应），两者职责不重叠。

## 踩坑点

- **`HEARTBEAT.md` 不要写时间型指令。**“每天 9 点发日报”放在心跳里会漂移，因为它只能靠 beat 落点去猜时间。这类需求全部迁到 cron。
- **`HEARTBEAT.md` 保持精简。** 每多一行，每个 beat 都多一份 token 开销；写成长文档还容易诱发误执行，只保留条件型条目。
- **cron 的 prompt 要自包含。** 独立会话没有主会话记忆，数据来源、文件路径、输出渠道都要写进 payload。
- **时区。** cron 表达式按 agent 运行环境时区解析，容器部署先确认 TZ，否则日报会“准时”发在错误的时间。
- **幂等设计。** heartbeat 可能因会话占用顺延，cron 重试也可能重复触发，任务要保证重复执行无副作用。

## 可复用建议

- 一句话选型：**cron 是闹钟，heartbeat 是巡逻。**
- 把 `HEARTBEAT.md` 当“条件触发器清单”，不要当日历用。
- 心跳间隔按需调大，非实时需求 30–60 分钟足够，直接省 token。
- 上线周期任务前，先跑一次性的 cron 验证投递链路（目标 channel 是否正确），再开启重复调度。
- 定期审计两个清单：cron 任务列表 + `HEARTBEAT.md`，删掉不再需要的条目，避免僵尸任务常年空转。

## 总结

cron 和 heartbeat 不是竞争关系，而是两种触发模型：**时间驱动走 cron，状态驱动走 heartbeat**。分清“什么时候做”和“看看有没有要做的”，定时任务的可靠性问题和 token 开销基本都能压下来。如果你有更复杂的组合实践（比如 cron 触发后回写心跳清单），欢迎在评论区交流。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/1ffbca854828e65a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/6ac3cbb5dafd226b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/cdd0b1a9536b4634.png)

