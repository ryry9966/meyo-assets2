---
title: Cron vs Heartbeat：OpenClaw 两种定时机制怎么选（附决策路径）
feedId: 40845
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景

OpenClaw 让 Agent「自己动起来」的机制有两种：**cron 任务**和 **heartbeat**。不少人在跑通第一个自动化之后都会遇到同一个问题：这两个都能定时触发，到底该用哪个？我在这两种机制上各踩过一些坑，把选择思路整理如下。

## 问题在哪

几个典型的误用场景：

- 想要「每天早上 9 点发日报」，结果写进了 HEARTBEAT.md 靠心跳碰时间，实际 9:37 才触发；
- 把重量级抓取脚本塞进心跳清单，Agent 每 30 分钟全量扫一遍，token 像流水一样走；
- 写了 cron 任务却指望它「记得昨晚聊过的内容」，跑起来发现它完全失忆。

根因是混淆了两种调度模型：

- **cron**：时间驱动，精确触发，独立会话，行为确定；
- **heartbeat**：间隔驱动，Agent 自主判断「要不要做、做什么」，复用主会话上下文。

## 做法：四步决策

**第一步：触发条件是精确时间点吗？**
是 → cron。日报、周报、定时提醒、一次性闹钟都归它。

**第二步：是「周期性看一眼状态，再决定做什么」吗？**
比如盯一个目录有没有新文件、巡检服务是否存活、整理收件箱——没有固定时刻，只有条件判断 → heartbeat。

**第三步：需要主对话的记忆吗？**
heartbeat 跑在主会话里，能看到你最近聊了什么；cron 每次是全新会话，需要的上下文要么写进 prompt，要么让它自己读文件。

**第四步：算成本。**
心跳每一次 beat 都消耗 token，空转也烧钱。轻量检查给 heartbeat，重活给 cron，或者 cron 去调脚本。

推荐一个混合模式：**cron 负责重活并把结果写入文件，heartbeat 清单只留一行「结果文件变了就通知我」**，各干各的。

配置要点：

```bash
# cron：每天 9 点日报
openclaw cron add --name daily-report \
  --cron "0 9 * * *" \
  --prompt "读取 workspace/notes.md，总结三条最重要的待办，推送到 Telegram"
```

heartbeat 只需在 Agent 工作目录放一个 HEARTBEAT.md：

```markdown
- 如果 downloads/inbox 有新文件，汇总后推送给我
- 如果服务清单里有进程挂了，立即告警
```

间隔在配置里调（默认 30 分钟，够用）。

## 踩坑点

1. **时区**。cron 表达式跟随网关主机的时区，海外 VPS 多为 UTC，「早上 9 点」会变成北京时间下午 5 点。改 TZ 环境变量，或干脆按 UTC 换算着写。
2. **HEARTBEAT.md 膨胀**。它是条件清单，不是脚本目录。超过十行就该迁移到 skill 或 cron 里。
3. **心跳频率激进**。从 30 分钟压到 5 分钟，token 成本翻六倍，而绝大多数 beat 是空转。先观察真实消耗再调频。
4. **cron 任务失忆**。每次运行是新会话，数据源、输出格式、投递渠道都要在 prompt 里写全，别省略。
5. **长任务阻塞心跳**。单次心跳运行超过间隔，下一拍会被跳过。长任务一律挪去 cron。

## 可复用建议

- 一句话判据：**cron 管确定性，heartbeat 管随机性**——前者看时间，后者看状态。
- 心跳清单模板：一行一个「如果 X，就 Y」，别写背景介绍。
- cron 的 prompt 当成独立工单写：输入、输出、投递目标缺一不可。
- 新任务先用默认频率上线，跑几天看 token 账单和触发命中率，再决定调频或迁移。

## 总结

两种机制不是二选一的竞争关系，而是分工：**固定时间做固定事，交给 cron；周期性看状态再做判断，交给 heartbeat**。想清楚「谁负责看表、谁负责看现场」，Agent 自动化的调度层基本就不会出大问题。遇到新需求时，先过一遍上面那四步，多数纠结会自动消失。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/e8e3ec7c50bb1811.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/6a71d9ddfbae3c79.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/9eab5f230f84eb95.png)

