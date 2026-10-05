---
title: cron 还是 heartbeat？OpenClaw 定时任务的分工实践
feedId: 40528
source: 综合讨论
publishedAt: 2026-10-05
---

## 背景

OpenClaw 的 agent 是常驻的：网关跑在 VPS 或家里的小机器上，7×24 活着。于是"让 agent 自己动起来"天然有两条路：

- **cron**：用 cron 表达式定点触发，到点拉起一次会话执行任务，可选把结果投递到某个聊天渠道；
- **heartbeat**：按固定间隔（默认约 30 分钟）唤醒主会话里的 agent，让它读工作区的 `HEARTBEAT.md` 自检一遍，没事就静默返回。

两者都能"定时"，但设计意图完全不同。我最早把所有定时需求都塞进 heartbeat，结果 token 消耗和主会话上下文一起膨胀；后来全改 cron，又发现"巡检收件箱、有值得回的再说"这种需要判断力的活它干不了。这篇记录最终的分工方式。

## 本质区别

| 维度 | cron | heartbeat |
|---|---|---|
| 触发逻辑 | 准点触发（cron 表达式） | 固定间隔轮询 |
| 会话 | 默认隔离会话，无历史 | 主会话，共享上下文 |
| 决策 | 执行写死的指令 | agent 自己读 HEARTBEAT.md 判断 |
| 成本 | 单次固定 | 间隔越短越烧 token |

一句话：**cron 是"到点干活"，heartbeat 是"定期看一眼有没有活"**。

## 我的做法

**准点交付类交给 cron**。晨报、周报、备份、定时提醒：

```bash
openclaw cron add \
  --name "morning-digest" \
  --cron "0 8 * * *" \
  --session isolated \
  --message "汇总昨晚通知与 RSS 新条目，输出 10 行以内摘要" \
  --deliver --channel telegram
```

关键在 `--session isolated`：每次运行都是干净上下文，便宜，也不污染主对话（具体参数以当前版本 `openclaw cron add --help` 为准）。

**需要判断的巡检类留给 heartbeat**。收件箱有没有值得看的东西、状态文件变没变、磁盘是否超阈值——这类事没有固定时刻。配置：

```json
{
  "agents": {
    "defaults": {
      "heartbeat": { "every": "45m", "target": "telegram" }
    }
  }
}
```

任务写在 `HEARTBEAT.md`，保持极简、写清触发条件：

```markdown
- GitHub 通知里出现需要我回复的内容才报告
- 磁盘使用率超 85% 才提醒
```

没动静时 agent 返回空转应答（HEARTBEAT_OK），不打扰你。

## 踩坑点

1. **heartbeat 间隔给太短**。5 分钟一次看着勤快，实际每次唤醒都是一次模型调用，一天近 300 次大多是空转。轻量巡检 30–60 分钟足够。
2. **把重活塞进 heartbeat**。它跑在主会话里，输出会进上下文。让它"抓取并总结 20 篇文章"，主对话很快变得又贵又迟钝。重活一律挪去 cron 隔离会话。
3. **cron 隔离会话没有记忆**。"总结新增内容"这种活，上次结果不落盘，这次就不知道"上次"是什么。做法：让任务把状态写进工作区文件（如 `state/last-digest.txt`），下次先读再对比。
4. **时区**。cron 表达式按网关机器时区解释，VPS 在 UTC，"早上 8 点"就不是你的 8 点。改机器时区或显式换算。
5. **错过不补跑**。网关离线期间的 cron 任务不会自动补执行，关键任务别拿它当唯一保障。
6. **投递静默失败**。`--deliver` 指向的渠道 token 没配好时，任务跑了、结果丢了，只有日志有痕迹。配完先手动触发一次验证。

## 可复用建议

- 准点交付、指令明确、产出要落渠道 → **cron**；
- 周期巡检、要上下文判断、可以静默 → **heartbeat**；
- 一条铁律：**状态写文件，别在会话里找记忆**。cron 靠文件传递状态，heartbeat 的 `HEARTBEAT.md` 本身就是状态；
- `HEARTBEAT.md` 控制在几行以内，只写"什么情况才报告"，其余全交给 cron。

## 总结

cron 和 heartbeat 不是二选一，而是分工：cron 负责确定性，heartbeat 负责感知力。要准点的事进 cron 隔离会话，要判断的事留给 heartbeat 定期自检，再用工作区文件打通两者的状态。这套组合在我这里跑了一个多月，token 消耗降下来了，该发生的事也一次没漏。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/656933813891cdd3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/bd78cb4f0bafc48e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/28fd95d344616f14.png)

