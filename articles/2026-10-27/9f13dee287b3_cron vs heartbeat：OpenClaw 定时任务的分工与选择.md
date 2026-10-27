---
title: cron vs heartbeat：OpenClaw 定时任务的分工与选择
feedId: 42766
source: 综合讨论
publishedAt: 2026-10-27
---

## 背景

OpenClaw 里有两种让 agent「按时干活」的机制，新人很容易混为一谈：

- **cron**：由网关内的调度器管理，用 cron 表达式定义触发时间，到点后拉起一次**独立**的 agent 运行，跑完按配置投递结果（发到频道、webhook 或仅存档）。
- **heartbeat**：网关周期性向**主会话**注入一次心跳 prompt，agent 自行判断要不要做事，无事则按约定保持沉默；任务清单通常写在 `HEARTBEAT.md` 里。

两者都能「定时」，但运行模型完全不同，选错会直接体现在 token 账单和消息噪音上。

## 问题

几个常见的模糊场景：

- 「每天早上八点半给我日报」——显然该 cron，但有人把它写进 HEARTBEAT.md，结果时间不准还持续空转。
- 「盯着 GitHub issue，有新回复就告诉我」——条件驱动，cron 硬轮询也行，但心跳里一句自然语言更省事，且主会话记得上下文。
- 混用后问题更隐蔽：心跳每次空转都是一次 LLM 调用；而 cron 任务若依赖「上次聊到哪了」，独立会话里根本拿不到。

## 做法 / 步骤

我的分流原则一句话：**时间驱动交给 cron，状态驱动交给 heartbeat**。

1. 把所有想让它定期做的事列出来，逐条标注：是「几点做」还是「有事才做」。
2. 固定时间、固定产出的任务进 cron：

```bash
openclaw cron add --name "daily-brief" \
  --cron "30 8 * * *" \
  --prompt "汇总过去24小时的仓库动态，输出不超过10行摘要" \
  --announce --channel telegram
```

   参数名随版本可能不同，以 `openclaw cron add --help` 为准。先用 `cron run <id>` 手动跑一遍，确认产出再等触发。
3. 监控、判断类任务进 heartbeat：把触发条件写进 HEARTBEAT.md，每条尽量是**可判定**的规则（如「有未处理的 CI 失败则通知我；否则不说话」），间隔从 30 分钟起步。
4. 两者需要配合时，用文件做交接：cron 把抓取结果写到工作区文件，心跳里只加一条「读该文件，有新增则汇报」。

## 踩坑点

- **心跳空转烧钱**：不写清楚「无事可做时的行为」，agent 每次 都会认真思考，甚至主动找话题。
- **用 heartbeat 追求准点**：心跳是「大约每 N 分钟」，网关忙时会排队，「9:00 整发」别指望它。
- **cron 会话失忆**：独立运行不带主会话上下文，prompt 必须自包含；需要历史就把状态落盘。
- **时区**：cron 表达式按网关所在时区解析，服务器在 UTC 时「早上八点」会差八小时，显式指定。
- **长任务堆叠**：耗时超过间隔会排队堆积，给任务配超时并保证幂等。
- **HEARTBEAT.md 腐化**：规则越加越多，agent 反而抓不住重点，定期修剪到一屏以内。

## 可复用建议

- 决策口诀：触发条件是「日历」还是「状态」？日历 → cron；状态 → heartbeat。
- 心跳是常驻值班员，cron 是闹钟：值班员适合巡逻，闹钟适合开会。
- 产出一律要求结构化（固定行数/字段），日后换投递渠道成本最低。
- 每周花五分钟 `openclaw cron list` 审一遍，删掉失效任务——定时任务的腐化速度比想象快。

## 总结

cron 和 heartbeat 不是二选一，而是分工：确定性调度交给 cron，需要上下文和判断的守望工作留给 heartbeat，中间用文件和明确规则衔接。把「时间驱动 vs 状态驱动」想清楚，成本和噪音都会明显降下来。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/9bb34bdcd3f83e2b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/25aed469d6477826.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/d09394d9994b2921.png)

