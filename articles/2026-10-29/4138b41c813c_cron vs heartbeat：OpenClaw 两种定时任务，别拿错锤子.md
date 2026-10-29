---
title: cron vs heartbeat：OpenClaw 两种定时任务，别拿错锤子
feedId: 42959
source: 综合讨论
publishedAt: 2026-10-29
---

## 背景

OpenClaw 用久了，迟早会遇到两类需求：一类是“每天早上 9 点把昨天的服务摘要发给我”，另一类是“有新东西的时候告诉我一声”。前者是打卡任务，后者更像巡逻。框架里对应两套机制：**cron** 和 **heartbeat**。很多人第一次配的时候会把它们混着用，跑两周才发现问题。

## 问题

典型的误用姿势有两种：

1. **把所有定时需求都塞进 heartbeat**：把间隔调到 5 分钟，prompt 写一大段。结果是 token 每轮都在烧，主会话上下文越来越长，agent 的话越来越多。
2. **用 cron 做需要“判断”的事**：比如想监控某个状态，拆成 8 个不同时间的 cron，逻辑其实是一个动态任务。

根子上是语义不同：**cron 是时间驱动——“到点了就干这个”；heartbeat 是状态驱动——“你现在去看一眼，有事再说”**。选错驱动方式，后面怎么调参都别扭。

## 做法

**cron 适合**：固定时间的确定性任务（日报、周报、定时提醒、周期清理）、一次性任务（三天后提醒我做某事）、需要精确控制投递渠道的场景。大致用法：

```bash
openclaw cron add --name daily-report \
  --cron "0 9 * * *" \
  --prompt "汇总昨天的 xxx，输出不超过 10 行" \
  --deliver --channel <目标频道>
```

cron 默认跑在隔离会话里，不会污染主会话，这一点对长期运行很重要。

**heartbeat 适合**：需要结合上下文做判断的巡逻类任务——检查收件箱、看 CI 状态、扫一眼有没有值得提醒的变化。关键是 prompt 要写清“查什么、什么条件才汇报”，没变化就静默（HEARTBEAT_OK），有事才开口。配置在 `openclaw.json` 里调 `heartbeat.every` 和对应 prompt 即可。

## 踩坑点

1. **heartbeat 间隔太短**。默认 30 分钟已经不便宜，改成 5 分钟，月底账单会教育你。高频轮询请放外部脚本，agent 只做判断层。
2. **heartbeat 里干重活**。它共享主会话上下文，塞大文件和长输出会稀释主会话，之后对话质量肉眼可见地下降。heartbeat prompt 控制在几句话内。
3. **cron 的时区和补跑**。先确认 gateway 的时区再写表达式；gateway 停机期间 cron 不会自动补跑，重要任务要配自查。
4. **重复检查**。同一件事 cron 和 heartbeat 都在管，就是双份开销、双份推送。agent“话多”之前，先查配置是不是重叠了。
5. **heartbeat 被要求“每轮给个简报”**。这等于关掉了静默机制，每次心跳都会发消息。无输出即静默，这个约定别破坏。

## 可复用建议

一句话决策法：

- 触发条件**能写成时间表达式** → cron
- 触发条件**必须看了才知道** → heartbeat
- 两者都行 → 优先 cron，可控、可审计、几乎不花 token

更进一步，监控类任务正确的分层是：**脚本负责轮询（传感器），agent 负责判断异常（裁判），cron 按需触发汇报（喇叭）**。把 agent 当传感器用，是最贵的用法。

## 总结

cron 是闹钟，heartbeat 是巡逻。闹钟精确、便宜、可验证；巡逻灵活、有上下文，但贵且行为不容易预测。拿到定时需求先分桶——“固定时间”还是“看了再定”——分对了，OpenClaw 会稳很多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/1da24a2984e35646.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/2a074158beeb84c9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/c44484ff3aadd22a.png)

