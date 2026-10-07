---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 40837
source: 综合讨论
publishedAt: 2026-10-07
---

## 背景

OpenClaw 里有两条"让 agent 定时干活"的路径：

- **cron**：经典定时调度器，用 cron 表达式或固定间隔触发一条 prompt，支持一次性任务、隔离会话，还能把结果投递到指定 channel；
- **heartbeat**：OpenClaw 特有的心跳机制，agent 按固定间隔被唤醒，去读 `HEARTBEAT.md`，有事就做，没事回一句 `HEARTBEAT_OK` 继续睡。

两者表面都能"定时跑任务"，设计意图却完全不同。

## 问题

新手常见的两类翻车：

1. 把所有定时需求都塞进 heartbeat，结果每隔半小时唤醒一次大模型，token 账单感人，且触发时机不精确；
2. 用 cron 硬做"巡检类"任务，每条 cron 都得维护一份自包含的长 prompt，缺了主会话记忆，写出来又脆又难改。

## 怎么定位：两个判断维度

我的经验是看**触发精度**和**是否依赖主会话上下文**：

- 时间点必须确定（工作日 9:00 发日报、周五提醒周报）→ cron
- "顺便看一眼"类（检查新文件、确认进程存活）→ heartbeat
- 结果要投递给别人看 → cron 配投递目标
- 需要结合近期对话记忆做判断 → heartbeat（它跑在主会话里）

## 具体做法

**cron 侧**：一条任务一个独立、自包含的 prompt。

```bash
openclaw cron add \
  --name "daily-digest" \
  --cron "0 9 * * 1-5" \
  --prompt "汇总昨晚仓库 issue，输出三条要点" \
  --deliver --channel telegram
```

要点：prompt 不假设"上一轮聊过什么"；输出走投递而不是指望翻日志；一次性任务用完即删。（具体参数以你版本的 CLI 为准。）

**heartbeat 侧**：把 `HEARTBEAT.md` 当收件箱。

```markdown
- 检查 ~/data/inbox 有无新文件，有就整理进 daily.md
- 确认 sync 进程存活，挂了先重启再通知我
```

要点：清单只放轻量、可失败、允许延迟几分钟的检查项；明确让 agent 没事时保持短输出。

## 踩坑点

- **时区**：Docker 里跑 OpenClaw，容器默认 UTC，cron 按 UTC 触发，"早上九点"实际是北京时间下午五点。显式设 `TZ=Asia/Shanghai`。
- **心跳唤醒成本**：间隔调到 5 分钟就是一天 288 次模型调用。没有强需求就保持默认间隔，高频任务交给 cron。
- **HEARTBEAT.md 写重活**：在里面放"生成完整报告"这种任务，每次心跳都会重跑。重活归 cron。
- **cron 输出无人消费**：任务跑了但投递目标没人看，等于静默失败。上线前先确认 channel 真的通。
- **会话污染**：cron 不做隔离时可能写进主会话上下文，把 heartbeat 的判断带偏。报告类任务建议用隔离会话。

## 可复用建议

一句口诀：**"几点做"用 cron，"要不要做"用 heartbeat。**

- cron 管"何时 + 做什么 + 结果发给谁"这条确定性链路；
- heartbeat 管"闲时巡检"，清单控制在 3 项以内；
- 两者可组合：日报由 cron 定时生成，heartbeat 巡检发现异常后再触发人工介入。

## 总结

cron 和 heartbeat 不是竞争关系，而是调度粒度的两级。把精确调度交给 cron，把上下文感知的巡检交给 heartbeat，token 成本和任务可靠性都会明显改善。如果你两边都在用，欢迎贴一下你的 `HEARTBEAT.md` 和 cron 配置，看看大家的分工方式。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/590267b23ce20665.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/5209a3f92333a645.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/4b1f6d480423758b.png)

