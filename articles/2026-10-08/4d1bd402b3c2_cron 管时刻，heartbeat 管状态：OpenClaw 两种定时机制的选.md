---
title: cron 管时刻，heartbeat 管状态：OpenClaw 两种定时机制的选型实践
feedId: 40879
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景

OpenClaw 里有两套定时触发机制，新人很容易混着用：

- **cron 任务**：标准 cron 表达式驱动，到点执行一个独立任务（跑一段 payload、调 skill、把结果投递到指定 channel）。
- **heartbeat**：主 agent 按固定间隔被"拍一下"，醒来后自己判断有没有事，没事就返回 `HEARTBEAT_OK` 继续睡。

两者表面上都能做"每 30 分钟检查一次 X"，但设计意图完全不同。

## 问题

我最初把所有周期任务都塞进 heartbeat，用一段很长的 prompt 让 agent 自己分辨该干嘛。结果：

1. token 消耗线性上涨——每次心跳都过一遍模型，哪怕 90% 的时间无事可做；
2. 时间敏感的任务（"工作日 9 点发日报"）经常晚点或漏发；
3. 日志里分不清是心跳例行检查还是真实触发，排障很痛苦。

后来反过来全用 cron，又遇到新问题：cron 是无状态单向的，"检查部署是否卡住，卡住才通知"这类条件逻辑要自己写判断、自己控降噪，很快难维护。

## 做法：按"时间驱动还是状态驱动"切分

一句话标准：**确定性时刻用 cron，持续性状态用 heartbeat**。

### 适合 cron 的

- 固定时刻的动作：早报、日报、定时备份、每周清理
- 需要投递到指定 channel 的单向任务
- 对精度有要求的（cron 精确到分钟，可配时区）
- 多个互不相关的小任务——各自独立，互不污染上下文

示意配置（字段以你版本的实际 schema 为准）：

```json
{
  "name": "daily-report",
  "schedule": "0 9 * * 1-5",
  "timezone": "Asia/Shanghai",
  "payload": "汇总昨日 commits 和待办，发到站会频道"
}
```

### 适合 heartbeat 的

- "有事才说话"的守护类任务：磁盘水位、异常日志、长任务是否卡死
- 依赖主会话上下文的判断——heartbeat 跑在主会话里，能利用之前的记忆
- 触发条件难以用时间表达、只能靠 agent 现场判断的场景

调优三点：

1. **prompt 写成否决式**：明确告诉 agent "默认无事，返回 HEARTBEAT_OK；仅满足条件 X 才行动"。这比罗列"要做的事"省得多。具体巡检项放进 `HEARTBEAT.md`，让它每次醒来读文件而不是读长 prompt。
2. **间隔宁长勿短**：从 30 分钟起步，确认有必要再加密。heartbeat 不是监控系统的替代品，秒级需求请外接真正的监控。
3. **心跳只读不写**：心跳里只做"读取 + 判断 + 必要时通知"，不要做有副作用的写操作。

## 踩坑点

1. **heartbeat 的 token 成本是隐性的**。默认间隔下一天几十次模型调用，叠加长会话上下文，一个月账单可观。先用当前模型配置跑一周看消耗再加需求。
2. **cron 失败是静默的**。投递失败、skill 报错，默认没人知道。关键任务要配"失败也通知"的兜底。
3. **时区**。cron 默认可能按 UTC 跑，"早 9 点"变下午 5 点，务必显式写 timezone。
4. **重叠执行**。执行时长超过间隔时，cron 可能堆积，heartbeat 会和主会话互相干扰。长任务两边都别放，外置处理。
5. **在心跳 prompt 里堆任务清单**是最常见反模式，又贵又慢还互相干扰。

## 可复用建议

- 选型口诀：**cron 管时刻，heartbeat 管状态**
- 触发条件能用 cron 表达式完整描述的，就别用 heartbeat
- heartbeat 只留一个，职责是"巡逻"；具体动作让它按需调 skill 或触发 cron
- 定期审计：拉两周执行日志，看 heartbeat 空转比例和 cron 静默失败数量

## 总结

两者不是竞争关系，是分工关系。cron 负责"日历上写好的事"，heartbeat 负责"需要盯着的事"。把确定性交给 cron，把判断力留给 heartbeat，token 账单和排障难度都会降一个档次。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/7e5ebe769a859525.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/9effa5bf774ce2c7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/3b97f03e46366b77.png)

