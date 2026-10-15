---
title: cron vs heartbeat：OpenClaw 两种定时机制的选型笔记
feedId: 41592
source: 综合讨论
publishedAt: 2026-10-15
---

## 背景

OpenClaw 里有两套"到点干活"的机制，刚上手很容易混用：

- **cron**：显式定时任务。用 cron 表达式或一次性时间触发，到点唤醒 agent，执行一段你预先写好的 prompt，结果投递到指定 session 或频道。
- **heartbeat**：心跳。按固定间隔（默认约 30 分钟）唤醒 agent，让它读工作区里的 `HEARTBEAT.md`，自行判断有没有事做；没事就返回 HEARTBEAT_OK，静默通过，不会打扰你。

两者的本质区别不在"能不能定时"，而在**由谁决定做什么**：cron 是你决定，heartbeat 是 agent 决定。

## 问题

两个典型翻车现场：

1. 把"每天 8 点整理日报"写进 HEARTBEAT.md 靠心跳兜底，结果 agent 某次心跳判断"今天没急事"就跳过了，日报三天两头漏。
2. 反过来，给十几条"每 5 分钟看一眼 XX"各建一个 cron，token 消耗翻着涨，半夜还在推送"一切正常"。

选错机制，轻则漏任务，重则烧 token、刷屏。

## 怎么选、怎么配

判断标准一句话：**有确定时间点的，用 cron；没有确定时间点、只是"盯着"的，用 heartbeat。**

### cron 适合

- 固定时刻的例行动作：早报、日报、每周汇总；
- 一次性提醒："明天 9 点提醒我复盘"（at 类任务，跑完即删）。

步骤：

1. 直接让 agent 用内置 cron 工具建任务，或走 CLI（具体参数以 `openclaw cron --help` 为准），填 name、表达式、prompt、投递目标；
2. 表达式先确认时区语义，容器里默认 UTC 很常见，写之前跑一下 `date`；
3. 一次性的事用 at 语法，别用 `*/5` 这类凑法。

### heartbeat 适合

- "盯着"类需求：目录有没有新文件、收件箱有没有待办、某个指标越没越界，有变化再说；
- 没有固定时刻，但希望 agent 定期自查。

步骤：

1. 配置里开启心跳并设间隔，例如 `agents.defaults.heartbeat.every: "30m"`，可加 `activeHours` 限定只在白天跑；
2. 工作区建 `HEARTBEAT.md`，写成可逐条判定的检查清单：检查什么、什么条件算"有事"、有事投递到哪；
3. 心跳唤醒时 agent 逐条核对，无事则静默。

一句话记忆：cron 的 prompt 是"到点了，做这件事"；heartbeat 的 prompt 是"到点了，看看有没有事"。

## 踩坑点

- **心跳间隔太短**：1–5 分钟一次等于高频唤醒 LLM，token 账单会教育你。15–30 分钟起步，观察一周再调。
- **HEARTBEAT.md 写成散文**：agent 判断"有没有事"的方差会很大。每条尽量可判定——有阈值、有明确的文件或状态可查。
- **cron 时区**：宿主机 UTC、人在 UTC+8，任务全在半夜跑。Docker 部署尤其常见，务必实测一次。
- **一次性任务忘了清理**：日历久了全是过期项。优先用 at 类任务，或定期 `cron list` 清点。
- **心跳清单里塞定时项**："每天 9 点做 X"靠心跳只能保证 9 点后的某次心跳执行，不保证 9 点整。定时的事交给 cron。

## 可复用建议

- 心里放一张分工表：**时间驱动 → cron；事件巡检 → heartbeat**。
- 心跳本质是轮询，解决的是"我不知道什么时候有事"。如果你其实知道时间，别用它。
- 投递分流：心跳产出的提醒发独立频道，别和日常对话混流。
- 定期复查两份"清单"：`cron list` 和 `HEARTBEAT.md`，不再需要的条目及时下线，任务列表和清单一样会腐烂。

## 总结

cron 和 heartbeat 不是竞争关系，而是分工：cron 管**确定性的时刻**，heartbeat 管**不确定性的事件**。我目前的配置是心跳 30 分钟 + 三条可判定的巡检项，其余全部走 cron。动手建任务之前，先想清楚"这次唤醒由谁决定做什么"，能省掉大部分折腾。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/7fa1a17423432674.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/f457315d412df6f5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/17c2d0fb29fd0da6.png)

