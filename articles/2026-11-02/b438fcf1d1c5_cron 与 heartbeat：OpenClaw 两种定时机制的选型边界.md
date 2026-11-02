---
title: cron 与 heartbeat：OpenClaw 两种定时机制的选型边界
feedId: 43283
source: 综合讨论
publishedAt: 2026-11-02
---

## 背景

OpenClaw 里有两套容易混淆的定时机制：`cron` 工具和 heartbeat。前者是显式调度器——你定义时间表、prompt、投递目标，到点就跑；后者是 agent 的周期性自检——每隔一段时间（默认 30 分钟）醒来一次，读工作区里的 `HEARTBEAT.md`，有事就办，无事静默跳过。

很多人第一次用会把 heartbeat 当成"简化版 cron"，然后发现它不准时、还会自作主张，反过来怀疑调度器不可靠。其实是两件事，适用场景不同。

## 问题

实际分界在哪里？踩过几个坑后我总结成一个判断标准：

- **cron 解决"时间驱动"**：必须在确定时刻发生的事。日报 09:00 发群、每 15 分钟拉一次数据、周五下班前提醒。
- **heartbeat 解决"状态驱动"**：不关心几点，只关心"现在有没有要处理的事"。收件箱有没有待汇总的消息、磁盘快满了没、有没有被遗忘的跟进项。

一句话判断：**能不能容忍 ± 一个心跳周期的延迟？不能，就用 cron。**

## 做法

**1. 硬性日程走 cron**

给独立 session，避免污染主会话：

```
cron add --name "晨报" \
  --cron "0 9 * * 1-5" \
  --session isolated \
  --prompt "汇总昨日消息，生成要点，投递到工作群"
```

isolated session 让每次执行在独立上下文里跑，结果通过 delivery 投递，主会话只收到结论、不收到过程。字段名以你版本的 `cron add --help` 为准。

**2. 弹性巡检走 heartbeat**

在配置里按需调整心跳周期，然后把当前要检查的事写进 `HEARTBEAT.md`：

```markdown
# Heartbeat
- 检查收件箱，有未回消息先汇总再提醒
- 磁盘使用率 > 85% 时通知我
- 无事则静默，不要发消息
```

心跳醒来读到实际内容才行动；文件里只有占位内容（如 `ok`）时直接静默跳过，不产生模型调用，这是它成本可控的关键。

**3. 组合使用**

我的日常分工：日报、周报、数据拉取全在 cron；heartbeat 只留两三条轻量巡检。heartbeat 发现重活时（比如日志该清理了），让它创建一个明确的 cron 任务或直接提醒我，而不是在心跳里自己干长任务。

## 踩坑点

- **用 heartbeat 当闹钟**。心跳间隔是"最长等待"而非精确时刻，实际触发会晚于预期。重要时间点一律 cron。
- **HEARTBEAT.md 写成大清单**。塞十几个条目，每次心跳都全量处理，token 翻倍还容易误动作。保持 1~3 个活跃项。
- **cron 任务跑在共享 main session**。输出刷屏、上下文被污染。除非任务需要与当前对话交互，否则用 isolated session。
- **时区**。cron 表达式按网关机器的本地时间解析，服务器在 UTC 时你写的"9 点"就不是你的 9 点。先确认时区。
- **停机窗口丢失**。网关重启期间到点的 cron 任务不会补跑。关键任务在 prompt 里让 agent 自查上次执行结果，或降低对单次执行的依赖。

## 可复用建议

1. 选型只问一句：能否接受 ±interval 的误差。
2. heartbeat 保持"轻"：条目少、动作小、明确"无事不打扰"。
3. cron 保持"隔"：独立 session、明确投递目标、命名清晰方便 `cron list` 管理。
4. 定期清理：把验证期留下的实验任务删掉，僵尸任务很占心智。
5. 重活下放：heartbeat 发现问题 → 转 cron 或通知人工，心跳本身不做长任务。

## 总结

cron 是日程表，heartbeat 是巡逻。前者保准时，后者保在线。把"必须几点发生"交给 cron，把"顺便看一眼"交给 heartbeat，预算和噪音就都可控了。选型错的表现也很直观：heartbeat 总是迟到，说明该用 cron；cron 列表越长越没人看，说明有些事其实只需要巡逻。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/8ef364443ebc685d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/35d4c26f04b9cfdf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/d0d8752f0e16e219.png)

