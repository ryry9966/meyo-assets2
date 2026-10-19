---
title: cron vs heartbeat：OpenClaw 两种定时任务的选型与踩坑
feedId: 41998
source: 综合讨论
publishedAt: 2026-10-19
---

## 背景

OpenClaw 里有两套让 agent 定时醒来的机制：

- **cron**：标准 cron 表达式调度，到点触发一次指定任务，可把结果投递到某个 channel；
- **heartbeat**：固定间隔（默认 30 分钟）的循环心跳，每次醒来由模型自己判断"有没有事要做"，没事就静默通过。

本质区别一句话：**cron 是"到点执行"，heartbeat 是"醒来巡逻"。**

## 问题

新手最常见的两种走偏：把所有定时需求都塞给 heartbeat，结果每天空转几百次 tick 白烧 token；或者反过来，用 cron 去做需要"看情况"的事，把判断逻辑硬编码进 prompt，改一次需求改一次配置。选错机制，成本和稳定性都会出问题。

## 做法

**第一步：给任务分类。** 两个维度——触发时间是否确定、执行内容是否依赖现场状态。

| 特征 | 选型 |
|---|---|
| 时间确定、内容固定（每天 8:30 日报、周日备份） | cron |
| 时间不定、内容依状态（收件箱巡检、跟进未完成任务） | heartbeat |

**第二步：固定事项交给 cron。** 思路如下（字段名以你版本的实际 schema 为准）：

```yaml
cron:
  jobs:
    - name: daily-report
      schedule: "30 8 * * *"
      prompt: "汇总昨天频道讨论，产出日报"
      deliver: true
      channel: telegram
      tz: Asia/Shanghai
```

**第三步：把 heartbeat 调便宜。** 三件事：

1. 间隔从默认 30 分钟拉长到 60 分钟起步，跑一周看日志再调；
2. 给心跳单独指定便宜的小模型；
3. 写一份极简心跳清单（如 `HEARTBEAT.md`），十几行以内，写清触发条件，并明确"没事返回 `HEARTBEAT_OK`、不要输出任何内容"的约定。

**第四步：组合而非二选一。** cron 准点产出日报，heartbeat 巡检时只核对"日报是否已产出、频道是否有未处理反馈"，让巡逻去验证结果，而不是重复生成。

## 踩坑点

- **心跳清单太长**：每次 tick 都带全量上下文，成本 = tick 次数 × 清单 token。清单务必压缩，细节放到按需读取的文件里。
- **心跳没有"不许说话"约束**：模型每次都热情输出，用户被消息轰炸。用 `HEARTBEAT_OK` 约定 + 数值化触发条件（阈值、开关）压住。
- **cron 时区**：容器内默认 UTC，日报会差 8 小时。显式设置 `TZ` 或任务级 tz。
- **长任务重叠**：上一次 cron 还没跑完，下一次又触发。加锁或按任务名去重。
- **heartbeat 做不了准点**：它是固定间隔轮询，不会按"周一 8 点"这种语义走，精确时刻一律交给 cron。
- **静默失败**：cron 跑挂了没人知道。失败也要投递到 channel，或至少落一份可查的日志。

## 可复用建议

- 一句话判断法：**能在日历上画出来的用 cron，画不出来、要看情况的用 heartbeat。**
- 先算成本再定间隔：cron 成本 ≈ 触发次数 × 单次 token；heartbeat 成本 ≈ tick 次数 × 判断 token。
- 两种机制都可能重试，动作设计成幂等可重入。
- 上线后跑 1~2 周，看日志和 token 消耗再调参，不要拍脑袋定间隔。

## 总结

cron 和 heartbeat 不是竞争关系，而是分工：确定性调度交给 cron，状态判断交给 heartbeat。把心跳清单压到最小、用 OK 约定压住输出、显式管好时区和幂等，这两套机制就能以很低的成本长期稳定地跑下去。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/96e1dd7a7dc7911e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/3bf2939209c3b98f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/860e79aa35d6e4ca.png)

