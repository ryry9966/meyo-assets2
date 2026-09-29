---
title: OpenClaw 的 cron vs heartbeat：两种定时机制怎么选
feedId: 39533
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

用 OpenClaw 做自动化一段时间后，几乎每个人都会撞上同一个问题：想让 agent「定期做点事」，该用 cron 还是 heartbeat？两者最终都是唤醒同一个 agent、跑同一个模型，但设计意图完全不同——cron 是时间驱动（time-driven），heartbeat 是状态驱动（state-driven）。选错了不是不能用，而是成本、噪声和可靠性都会明显变差。

## 问题：两种典型误用

第一种是把 heartbeat 当任务队列用。所有周期性需求都塞进 HEARTBEAT.md，agent 每次唤醒都要把整个清单在脑子里过一遍，no-op 唤醒越来越多，token 消耗和消息噪声一起涨。

第二种是全用 cron 硬编码。把「检查依赖有没有更新」「有异常就处理」这类需要临场判断的事拆成一堆定时任务，结果 job 列表膨胀、每个 prompt 越写越长，而真正需要判断的事反而没人管。

判断标准其实只有四条：

- 触发源是明确的时刻，还是「醒来后看状态再决定」？
- 时间精度要分钟级，还是「每隔一段时间」就行？
- 要不要 agent 自主判断做不做、做多少？
- 对单次唤醒成本的敏感度如何？

## 做法：按触发源分流

**时间驱动 → cron。** 凡是「知道几点干」的事都归 cron：每天 09:00 的日报、每周一的巡检、每隔 2 小时抓一次数据。示意如下（参数以 `openclaw cron add --help` 输出为准）：

```bash
openclaw cron add \
  --name "ops:daily-report:09am" \
  --cron "0 9 * * *" \
  --tz Asia/Shanghai \
  --message "读取 ~/logs 下昨天的报错日志，汇总 Top 5 问题，生成日报发到运维群" \
  --session isolated \
  --delivery announce
```

三个要点：显式指定时区；session 用 isolated，别污染主对话上下文；message 要自包含——isolated 会话没有主对话记忆，写一句「帮我看看情况」只会得到一次质量很差的回答。

**状态驱动 → heartbeat。** 正确用法是「值班巡检」：HEARTBEAT.md 写成短 checklist（建议不超过 10 条），每条给出明确的判定条件和「无事可做时返回 HEARTBEAT_OK」的出口。agent 每次醒来自己判断：没事就静默，有事才处理并推送。它适合「不知道要不要做、只知道要常看看」的场景，比如盯一个长任务的进度、定期确认某个服务还活着。

**混合模式**是我实际用下来最稳的结构：cron 负责确定性动作，heartbeat 负责巡检和分发——发现异常后要么直接处理，要么动态建一个临时 cron job 跟进，事后清理。

## 踩坑点

- heartbeat 间隔设太短。它不是秒级轮询工具，每次唤醒都是一次完整的模型调用，no-op 也要花钱。
- HEARTBEAT.md 写成散文。条件模糊，agent 的判断就会抖动，表现为「有时做有时不做」。
- cron 忘了设时区。服务器多半是 UTC，任务整体偏移 8 小时，日报变成下午茶。
- cron 用了 main session。定时输出混进主对话上下文，日常对话质量被稀释。
- no-op 判定不明确。清单里没写「无事返回 HEARTBEAT_OK」，agent 每次都「觉得有事」，频道被刷屏。

## 可复用建议

- 选型口诀：知道几点干 → cron；不确定要不要干、只知道常看看 → heartbeat。
- 一个任务一个 cron job，命名用 `域:动作:频率`，方便 list 和清理。
- HEARTBEAT.md 里的高频巡检项一旦稳定，就下沉为 cron job + 告警，heartbeat 只留真正需要判断的项。
- 新任务先在 heartbeat 里试运行，验证任务描述的质量，跑顺了再固化为 cron。
- 定期看「唤醒次数 vs 实际产出」的比例，no-op 率过高就降频或精简清单。

## 总结

cron 是排班表，heartbeat 是值班员。前者精确、便宜、但僵硬；后者灵活、有判断力、但每一步都有成本。不必二选一——用 cron 承接确定性，用 heartbeat 兜住模糊性，把 agent 省下来的注意力留给真正需要判断的地方。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/41e0cdbeaa444e7f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/da00e7e671497c98.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/910432910fcf9e6d.png)

