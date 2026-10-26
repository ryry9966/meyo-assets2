---
title: cron 还是 heartbeat？OpenClaw 两种定时机制选型笔记
feedId: 42648
source: 综合讨论
publishedAt: 2026-10-26
---

## 背景

OpenClaw 里想让 agent「定时干活」，官方给了两条路：内置 cron 调度和 heartbeat。两者都能周期性触发 agent，但语义完全不同。社区里常见的两类翻车：一是拿 heartbeat 干精确调度的活，结果时间点飘得没法用；二是拿 cron 模拟巡检，一分钟一次 agent turn，token 账单直接起飞。这篇把选型逻辑理一遍。

## 两者的本质区别

**cron 是时间驱动**：到点就跑，跑完即止。支持一次性任务和 cron 表达式循环，触发时间确定性高。适合「每天 9 点发日报」「周三下午提醒我」「凌晨 2 点跑一次批处理」这类有明确时间语义的任务。

**heartbeat 是状态驱动（值班式）**：agent 每隔固定间隔被唤醒一次，扫一遍 workspace 里的 `HEARTBEAT.md` 关注清单，没事就回 HEARTBEAT_DONE 静默通过，有事才推给你。它不承诺「几点做某事」，只承诺「我会定期看一眼」。

一句话：cron 是闹钟，heartbeat 是值班员。

## 做法

1. **先问一个问题**：这个任务有没有精确的时间语义？有 → cron；只是「帮我盯着，有情况叫我」→ heartbeat。
2. **定点任务走 cron**：用 CLI 添加任务，指定触发表达式、消息内容和投递渠道。优先用内置 cron 而不是系统 crontab 调 CLI——内置任务自带投递链路和会话管理，失败也好排查。
3. **巡检类走 heartbeat**：把关注事项写进 `HEARTBEAT.md`，条目化、可判定，比如「检查收件箱有没有标星未读」「磁盘超过 85% 就报」。间隔用 heartbeat 的间隔配置调（默认 30 分钟），推送目标单独指定。具体参数名以你版本文档为准。
4. **混用才是常态**：heartbeat 做常驻巡检兜底，cron 挂定点强任务。定期 `openclaw cron list` 审计一次性任务，跑完就删，别留僵尸。

## 踩坑点

- **heartbeat 间隔别低于 10 分钟**。每次唤醒都是一次完整 agent turn，间隔越短，token 成本和误报噪声同步上涨。
- **时区**。cron 表达式按网关所在环境的时区解释，VPS 多半是 UTC，「每天 9 点」会差出 8 小时。部署后先跑一个低风险任务验证。
- **`HEARTBEAT.md` 写成「帮我盯着点」这种模糊指令**，agent 每次自行发挥，行为不稳定还容易漏。写成可判定的检查项。
- **cron 的 message 别塞大段上下文**。每次触发都按 turn 计费，长上下文放文件里让它自己读。
- **heartbeat 的推送目标慎选群聊**。「无事静默」是设计预期，但一旦它判定有事，消息就进群了。

## 可复用建议

- 选型口诀：有「几点做」，用 cron；只有「帮我看」，用 heartbeat。
- 不要用高频 cron 模拟 heartbeat，那是拿闹钟当值班员，成本和噪声双输。
- heartbeat 清单的维护思路和 SRE 告警规则一样：每条可判定、有优先级、低价值项定期清掉。
- 关键 cron 任务加上投递确认，别把「任务跑了」和「结果送到了」混为一谈。

## 总结

cron 和 heartbeat 不是二选一的竞争关系，而是分工：时间语义归 cron，状态语义归 heartbeat。判断标准只有一个——这个任务在乎的是「几点」，还是「有没有事」。想清楚这一点，两种机制都能各司其职，你的 agent 才是一个安静的同事，而不是一台乱响的机器。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/02054f0efe699993.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/dbf1603e282aefec.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/df6aca63b261e50e.png)

