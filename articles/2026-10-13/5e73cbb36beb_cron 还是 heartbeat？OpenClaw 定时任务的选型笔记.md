---
title: cron 还是 heartbeat？OpenClaw 定时任务的选型笔记
feedId: 41374
source: 综合讨论
publishedAt: 2026-10-13
---

## 背景

跑过一段时间 OpenClaw，迟早会遇到同一个需求：让 agent 在固定节奏下自己干活。平台给了两条路：**cron** 与 **heartbeat**。前者是确定性调度；后者是周期性心跳——每个 tick 把 agent 叫醒，对照工作区的 HEARTBEAT.md 自查"现在有没有该做的事"，没有就回 HEARTBEAT_OK 静默结束。两条路都能实现定时任务，常见状态是全押一边，然后各自踩坑。下面是我翻车后整理的选型结论。

## 问题

两个真实案例：

1. "每天早上 9 点发日报"被写进 HEARTBEAT.md，靠模型自己判断时间。结果发送点在 8:40–9:50 之间漂移，且每个 tick 都要为这条规则付一次判断成本。
2. "收件箱有新邮件就整理摘要"被配成每 5 分钟一次的 cron。新邮件一天来不了几封，95% 的运行是空转，token 和日志全是噪音。

本质区别一句话：**cron 回答"什么时候做"，调度责任在配置；heartbeat 回答"要不要做"，判断责任在模型。** 把"准时"交给模型，或把"判断"交给 cron，都会别扭。

## 做法

**用 cron：时间点明确、动作幂等。** 日报、日历同步、每日备份都属于这类。用 CLI 注册：

```bash
openclaw cron add --name daily-report \
  --cron "0 9 * * *" \
  --message "汇总昨天全部仓库的 commit 与 PR，生成日报发到 #dev"
```

表达式由 gateway 精确触发，不依赖模型的时间感知。

**用 heartbeat：触发条件依赖上下文、时间不敏感。** 巡检收件箱、盯目录异常、检查有无失败任务需要补救，都属于这类。检查项写进 HEARTBEAT.md，每条保持"条件 + 动作"格式，整个文件控制在十几行内，让一次自查尽量便宜。

**组合是当前主流方案：** cron 管"准时的事"，heartbeat 管"兜底巡检"。例如日报由 cron 触发，heartbeat 每半小时检查一次"昨天的日报是否发送失败，失败则补发"。

## 踩坑点

- **heartbeat 间隔别低于 15 分钟。** tick 是持续成本，我曾调到 5 分钟，日消耗肉眼可见地涨，新增价值几乎为零。
- **HEARTBEAT.md 里不写时间敏感规则。** 模型的时间感知不可靠，精确时间一律交给 cron。
- **cron 有时区坑。** gateway 按宿主机配置解释表达式，服务器在 UTC 时，`0 9 * * *` 其实是北京时间 17 点。部署后先跑一个一次性 job 验证。
- **heartbeat 检查项要幂等。** tick 可能与手动会话重叠，执行动作前先验证条件仍成立，比如补发日报前先确认确实没发出去。
- **cron 失败是静默的。** 关键 job 要在 message 里要求输出可校验的结果，或用 heartbeat 做对账。

## 可复用建议

- 选型口诀：需求能写成 "at X do Y" 就用 cron；只能写成 "when Z seems needed" 就用 heartbeat。
- 每次想往 HEARTBEAT.md 加条目，先问"这条能不能变成 cron"。能变就变，心跳清单越短越好。
- 心跳是持续成本，cron 是事件成本，预算敏感时优先 cron。
- 上线后观察一两周 tick 日志再调间隔，比拍脑袋定参数靠谱。

## 总结

cron 是确定性时钟，heartbeat 是代理式巡检，二者不是替代关系。选型的核心不是谁更强，而是把判断权交给谁：**时间交给配置，判断交给模型**。各归其位，agent 才省心，token 也省心。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/7b8823141bf9c7ca.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/0f88d58db78556d3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/2d9f60faf2e39e3f.png)

