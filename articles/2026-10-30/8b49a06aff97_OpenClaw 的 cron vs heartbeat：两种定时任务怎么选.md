---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 43022
source: 综合讨论
publishedAt: 2026-10-30
---

## 背景

OpenClaw 里给 agent 加"自动性"有两条路：**cron** 和 **heartbeat**。cron 是精确的时间触发——到点就按预设 prompt 跑一次；heartbeat 是周期性唤醒——每隔 N 分钟把 agent 拉起来，让它自己判断有没有事要做。两者都能实现"没人盯着也能干活"，但语义完全不同，混用是社区里最常见的翻车来源之一。

## 问题

典型的错误用法有两类：

1. 用 heartbeat 做"每天 9 点发日报"——heartbeat 只保证频率不保证时刻，9 点的任务可能 9:23 才执行；
2. 用 cron 做监控类任务——"每小时检查有没有异常"，但异常不一定整点出现，cron 只会让你在固定时刻假装在看。

本质区别一句话：**触发条件是"时间"还是"条件"。**

## 做法

建议按这个流程拆：

1. **盘点任务**，分成两类：时刻确定的（日报、同步、备份）和条件驱动的（盯部署、等回复、巡检）。
2. **时刻确定的走 cron**，prompt 写成命令式、自包含：做什么、输出到哪、失败怎么办，一次说清，不给 agent 自由发挥的空间。
3. **条件驱动的走 heartbeat**，间隔按"能容忍的最大延迟"定，默认 30 分钟起步。heartbeat prompt 保持极简，并明确告诉 agent：没事就简短确认、保持沉默，避免每次唤醒都长篇输出。
4. **组合使用**：cron 负责确定性的主线任务，heartbeat 当兜底看门狗——它不直接干活，只负责发现"该干活了"然后推进。

## 踩坑点

- **heartbeat 间隔太短**：5 分钟一次，token 消耗直接翻倍，而且 agent 容易"为了显得有用"编任务出来。宁可间隔长一点。
- **cron prompt 太模糊**："帮我看看数据"这种 prompt 会让 agent 醒来后先花一大段推理决定自己要干嘛，又慢又贵。
- **时区问题**：cron 按网关机器的时区走。服务器在 UTC、你在 UTC+8，日报会准时地晚八小时。
- **任务重叠**：上一个 heartbeat 任务没跑完，下一次唤醒又来了。加个状态标记或文件锁防重入。
- **静默堆积**：cron 任务只增不减，没人审计。每月清一次，把无人认领的删掉。

## 可复用建议

- 判断口诀：**"什么时候做"清楚 → cron；"要不要做"需要判断 → heartbeat。**
- 面向人的定时输出（日报、摘要）一律 cron，别赌 heartbeat 的时刻精度。
- 给每个定时任务记一行元信息：负责人、上次运行时间、上次 token 消耗。审计永远比补救便宜。
- 笔记本或会休眠的机器跑 heartbeat 不可靠，关键任务要么放到常驻环境，要么改成外部事件触发（webhook、CI 回调），别硬凑周期。

## 总结

cron 和 heartbeat 不是竞争关系，是分工关系：cron 是确定性的调度器，heartbeat 是带判断力的看门狗。按"时间触发还是条件触发"来选，再配上克制的 prompt 和定期审计，定时自动化才能长期稳定地跑下去。欢迎在评论区贴出你自己的任务清单，看看大家都是怎么分的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/0db8b1853d5f2000.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/07e840798fb06c37.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/bfec9e00b896a993.png)

