---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 37513
source: 综合讨论
publishedAt: 2026-09-14
---

# OpenClaw 的 cron vs heartbeat：两种定时任务怎么选

## 背景

OpenClaw 里有两套"让 agent 自动动起来"的机制，刚上手很容易混淆：

- **cron**：内置调度器（`cron` 工具，任务存在 `~/.openclaw/cron/jobs.json`），支持一次性触发、固定间隔、标准 5 段表达式加时区。任务可以是独立的 agent turn（自己的会话和参数），也可以只作为 system event 注入某个会话。
- **heartbeat**：agent 级周期心跳，默认每 30 分钟向主会话注入一条心跳消息。agent 醒来看一眼工作区的 `HEARTBEAT.md`，有事做事，没事走轻量确认路径，几乎零成本。

## 问题

两个真实踩坑场景：有人把"每天 9 点发日报"挂在 heartbeat 上，结果前一晚 agent 在跑长任务，心跳顺延，日报 10 点半才出；也有人把十几条巡检全建成了 cron 独立会话，主会话反而丢了"今天已经汇报过"的记忆。本质是没分清两者的定位。

## 怎么选与做法

**选 cron 的信号**：

1. 有确切时刻或日历语义（"工作日 9 点""每周一"）；
2. 需要一次性提醒（at 触发）；
3. 结果要推送到 Telegram/Slack 等 channel；
4. 任务不该污染主会话上下文 → 用 isolated session。

配置示意（参数名以你手头版本的 `cron add` 帮助输出为准）：

```bash
cron add --name "morning-digest" \
  --cron "0 9 * * 1-5" --tz "Asia/Shanghai" \
  --session isolated \
  --message "汇总昨晚 GitHub 通知，输出不超过 5 条的中文摘要" \
  --deliver --channel telegram
```

**选 heartbeat 的信号**：

1. 任务是周期巡检/收尾，早十分钟晚十分钟无所谓；
2. 依赖主会话上下文（知道今天聊过什么、干过什么）；
3. 一堆零散小事不值得逐条建 cron job → 全写进 `HEARTBEAT.md`。

步骤：在 `openclaw.json` 设置 `agents.defaults.heartbeat.every`，工作区建一个 `HEARTBEAT.md`，只写 3–6 行清单；文件清空或删除时心跳走近乎零开销的确认路径。

## 踩坑点

1. **heartbeat 不是精确调度**。它是"每隔 N 分钟尝试唤醒"，会话忙时会顺延，需要准点的必须用 cron。
2. **时区**。cron 表达式默认按宿主机时区解释，跨时区部署一定显式传 tz，否则 9 点的任务会在错误的时间触发。
3. **`HEARTBEAT.md` 无限膨胀**。心跳每次都把这份清单带进上下文，写到 30 行就开始烧 token，只保留高价值条目。
4. **cron 会话选错**。注入主会话的 system event 适合"提醒主 agent"，独立跑批用 isolated，混用会互相污染上下文。
5. **停机不补跑**。gateway 挂掉期间错过的触发会被跳过，关键任务要监控守护进程本身的存活，别指望任务自愈。
6. **调试顺序**：先看 `cron list` 里 job 的 lastStatus 和 `cron runs` 的历史，确认是投递失败还是 prompt 问题，再动手改。

## 可复用建议

一句话分工：**cron 管"计划"，heartbeat 管"习惯"**。

- 时刻敏感、要推送、要隔离 → cron；
- 上下文敏感、容忍漂移、碎片化清单 → heartbeat；
- 两者可以组合：heartbeat 负责巡检中发现新事项，再动态注册一次性 cron job 执行"到点提醒"，避免常驻高频任务。

## 总结

用三个问题过一遍：要不要准点？要不要推送到 channel？要不要主会话记忆？答案基本自然浮现。别把 cron 当万能调度器，也别把 heartbeat 当 cron 的穷人版——它们是互补的两种心智模型，用对位置比堆功能更重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/47e046f399541d06.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/18791d457ed0b735.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/679e20841f13914c.png)

