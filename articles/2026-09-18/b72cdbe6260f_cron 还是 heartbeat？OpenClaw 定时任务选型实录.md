---
title: cron 还是 heartbeat？OpenClaw 定时任务选型实录
feedId: 38045
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

OpenClaw 里有两套容易被混淆的"定时"能力：

- **cron**：用 cron 表达式触发，`openclaw cron add` 创建，默认在隔离会话里运行，执行完把结果投递到指定 channel（也可以不投递，只跑动作）。
- **heartbeat**：agent 每隔固定间隔自动醒来一次，读取工作区的 `HEARTBEAT.md`，自己判断这轮有没有事要做。没事就回一个 `HEARTBEAT_OK`，网关静默吞掉，不打扰任何 channel。

两者的共同点是"都会周期性唤醒 agent"，但设计意图完全不同：cron 本质是日程表，heartbeat 本质是巡逻哨。

## 问题

社区里我见过两种典型误用：

1. **全塞给 heartbeat**。把"每天 9 点发日报"也写进 HEARTBEAT.md，结果 agent 每半小时醒一次，醒来先判断"现在是不是 9 点"——判断逻辑写在自然语言里，偶尔误判，token 也白白烧掉。
2. **全用 cron**。把"盯一下 CI 有没有挂"写成每 10 分钟一次的 cron，但每次都是隔离会话，agent 不知道之前查到哪一步，每次从零开始，输出无法连续。

本质区别一句话：**cron 解决"什么时候做"，heartbeat 解决"要不要做"。**

## 做法

我的判断流程是三个问题：

1. 触发时间是确定的吗？确定 → cron。
2. 结果需要主动投递到 channel 吗？需要 → cron（配 `--deliver`）。
3. 是"条件满足才行动"的巡检吗？是 → heartbeat。

**cron 适合确定性动作**，比如每天早报：

```bash
openclaw cron add --name "morning-digest" \
  --cron "0 9 * * *" \
  --message "汇总过去24小时：日历、未读消息、待办" \
  --deliver --channel telegram
```

**heartbeat 适合条件性巡检**，把 `HEARTBEAT.md` 写成短清单：

```markdown
# HEARTBEAT.md
- 检查 ~/sync/inbox 是否有新文件，有则归类并通知我
- 检查 CI 状态页，仅在失败时报告
- 没有事做时直接 HEARTBEAT_OK，不要说话
```

两者可以组合：cron 负责"到点必做"的动作，heartbeat 负责巡逻，发现问题后再触发具体技能去处理。

## 踩坑点

- **heartbeat 间隔别贪短**。默认 30 分钟是合理的。我试过 5 分钟，token 消耗翻了几倍，收益几乎为零。
- **HEARTBEAT.md 别写成散文**。条目要动作明确，并显式写上"无事保持沉默"，否则 agent 会找理由汇报点什么。
- **cron 的时区**。调度时间按宿主机时区解释，服务器在 UTC 上时，早 9 点会变成下午 5 点。要么显式指定时区，要么部署后先挂一个每分钟一次的测试任务验证。
- **cron 拿不到主会话记忆**。隔离会话意味着 agent 是"失忆"的，需要的上下文要写进 payload，或让它去读指定文件。
- **HEARTBEAT_OK 不是故障**。日志里大量出现是正常的，表示"这轮没事"，别当报错处理。

## 可复用建议

1. 一句话选型：**知道"几点做"用 cron，只知道"什么条件做"用 heartbeat**。
2. `HEARTBEAT.md` 控制在 3–5 条以内，条目越多判断越不稳定。
3. 心跳发现的问题，处理动作沉淀成 skill——心跳只负责"发现"，不负责"执行"。
4. cron 任务命名带前缀（`daily-`、`weekly-`），`openclaw cron list` 排查时一眼能懂。

## 总结

cron 和 heartbeat 不是二选一，而是分层：把确定性交给 cron，把不确定性交给 heartbeat，自动化才能既省 token 又可靠。如果你现在的 `HEARTBEAT.md` 里还躺着一条"每天 X 点做 Y"，今天就可以把它迁出去。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/404e0489d661f9b1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/b3c0e3ba9f72bec5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/c192940dfb25f0c5.png)

