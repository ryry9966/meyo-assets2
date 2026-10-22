---
title: cron 还是 heartbeat？OpenClaw 两种定时机制的分工
feedId: 42329
source: 综合讨论
publishedAt: 2026-10-23
---

## 背景

OpenClaw 有两套让 agent 定时干活的机制，新人经常混着用：

- **Heartbeat（心跳）**：agent 按固定间隔醒来（默认约 30 分钟，`agents.defaults.heartbeat.every` 可调），读一遍工作区的 `HEARTBEAT.md`，有活干活，没活回一个极简 ack 继续睡。
- **Cron（定时任务）**：用 `openclaw cron add` 注册的确定性任务，支持 cron 表达式、时区、投递渠道和独立会话，重启不丢。

两者本质都是"到点唤醒 agent 跑一段 prompt"，但语义完全不同：heartbeat 是巡更，cron 是闹钟。

## 问题

社区里两种典型误用：一是把所有定时需求都塞进 HEARTBEAT.md，心跳 token 开销越来越高，任务还不准点；二是把"大概看一眼就行"的巡检拆成一堆 cron job，改个频率还要动命令行，管理成本上去了。

## 做法

**有明确时间点的，交给 cron。** 比如每天早上推日报：

```bash
openclaw cron add --name daily-digest \
  --cron "30 8 * * *" --tz Asia/Shanghai \
  --session isolated \
  --prompt "汇总今天日程和未读消息，整理成 5 条以内的摘要，发到我的 Telegram"
```

要点：`--session isolated` 让任务在独立会话里跑，不污染主对话；结果投递到指定渠道；具体参数名以 `openclaw cron add --help` 输出为准。注册完先 `openclaw cron run <id>` 手动触发一次，确认链路通了再等准点。

**没精确时间要求、只是定期看一眼的，写进 HEARTBEAT.md：**

```markdown
# HEARTBEAT.md
- 检查 ~/logs/error.log 是否有新增 ERROR，有就告诉我
- 有超过 2 小时未回的重要消息就提醒我
- 没事别说话
```

heartbeat 的优势是便宜：空转时几乎零成本；改清单就是改文件，下个心跳自动生效，不用碰 CLI。

一句话选型：**要准点 → cron；要随缘 → heartbeat。**

## 踩坑点

- **cron 时区**：默认跟宿主机走。服务器在 UTC，"每天 8 点"就变成北京时间 16 点，显式加 `--tz`。
- **HEARTBEAT.md 涨肥**：清单每个心跳都会被完整读一遍，一长、或塞进需要长上下文的任务，开销和延迟立刻上来。控制在十行以内，干完的条目让 agent 主动划掉。
- **独立会话没记忆**：isolated 任务读不到主对话历史，prompt 必须自包含——路径、口径、输出格式都写全。
- **wake 模式选错**：cron 选"跟随下个心跳"时，实际触发是心跳时刻而非计划时刻，准点任务别用这个模式。
- **心跳间隔别贪快**：5 分钟一次等于让模型全天待命，费用和限流都不划算，除非确有近实时监控需求。

## 可复用建议

- 把 HEARTBEAT.md 当"巡检看板"而非"任务队列"：短、平、可勾销。
- 发现任何带固定时刻的条目，随手迁进 cron。
- cron 报告类任务统一 isolated + 指定投递渠道，prompt 末尾加一句"无异常就不发消息"，能省掉大量噪音推送。
- 新任务上线先手动触发验证，再用 `openclaw cron list` 和心跳日志核对实际执行时间，最后才调间隔。

## 总结

heartbeat 和 cron 不是二选一，而是分工：cron 负责"日历上写得出来的事"，heartbeat 负责"顺手看一眼的事"。判断标准就一条——这件事需不需要准点发生。需要，cron；不需要，heartbeat。各归各位之后，OpenClaw 的定时自动化才能既准时又省钱。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/10b9b673cd9a8d3b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/dc64d88b0cea1cf8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/3a634283295690ee.png)

