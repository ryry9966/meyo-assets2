---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 38573
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

OpenClaw 里跑的是常驻 agent，“让它定时干活”有两条路：**cron** 和 **heartbeat**。两者本质上都是定时唤醒 agent 执行一轮任务，但设计意图完全不同。我最早把它们混着用，结果既花了多余的 token，又错过了准点触发的需求。这里把机制差异和选型思路整理一下。

## 两者的机制差异

**cron：显式调度。** 你或 agent 通过 cron 工具注册任务，每条任务包含 schedule（once / 固定间隔 / cron 表达式）、payload（要执行的 prompt）、session（main 或 isolated）和可选的投递目标。任务持久化在本地 jsonl 文件里，gateway 重启后仍然生效。到点触发、做完收工，时间可控。

**heartbeat：周期自检。** 按配置间隔（默认约 30 分钟，可调）唤醒 agent，让它读 workspace 下的 `HEARTBEAT.md` 清单，自行判断有没有值得动手的事；没事就回一个简短的 HEARTBEAT_OK 结束本轮。可以用 activeHours 限制只在某段时间心跳。

一句话概括：**cron 是“到点做指定的事”，heartbeat 是“定时自检，有事再说”。**

## 怎么选：三步走

1. **先分类需求。** 有明确时间点、明确动作的——每天 9 点发日报、每周清理临时目录、定时提醒开会——全走 cron。模糊的条件监控——“留意 GitHub 通知”“有人回了没跟进的线程就提示我”——交给 heartbeat。
2. **配 cron 时盯三个字段。** 对 agent 说清“什么时间、做什么、结果发到哪”，让它建任务；重点确认 schedule 是否符合预期、session 选 main 还是 isolated、delivery 是否可用。重的、产出型任务建议 isolated，避免污染主会话上下文。
3. **配 heartbeat 从清单入手。** 先在配置里定间隔和活跃时段，再把持续关注的事项写进 `HEARTBEAT.md`，一行一条、按优先级排。心跳的质量取决于清单的质量。

两者不互斥：cron 管确定性动作，heartbeat 留作兜底自检，是我现在的默认架构。

## 踩坑点

- **拿 heartbeat 做精确提醒。** 30 分钟间隔意味着最多迟到半小时，还可能被 activeHours 截掉。时间敏感的任务一律 cron。
- **心跳频率往低了调。** 每次心跳都是一轮完整调用，哪怕回 HEARTBEAT_OK 也花 token。5 分钟一次就是一天 288 轮，成本先算清楚。
- **`HEARTBEAT.md` 越堆越长。** 清单越长，每轮判断越慢、误触发越多。我控制在 10 行以内，每周清一次。
- **cron 的 isolated session 没有主会话记忆。** prompt 必须自带上下文，否则任务执行得很“失忆”；需要延续对话的任务才用 main。
- **时区。** Docker 容器默认 UTC，“早上 9 点”实际是下午 5 点。cron 和 heartbeat 的 activeHours 都受影响，部署时显式设 `TZ`。
- **任务重叠。** 上一轮没跑完下一轮又触发，给长任务设超时、间隔留余量。

## 可复用建议

- 选型口诀：**时间点给 cron，状态判断给 heartbeat。** 高频且确定性的检查（比如每 5 分钟探一个接口），用短间隔的 isolated cron job，而不是压缩心跳。
- 把 `HEARTBEAT.md` 当“值班手册”维护，而不是任务队列；临时需求先走 cron，跑稳了再考虑要不要进心跳清单。
- 上线后先观察几天 token 消耗，再回头调心跳间隔和清单长度。
- 凡是“必须在某个时刻发生”的事，不要托付给 heartbeat。

## 总结

cron 和 heartbeat 不是二选一，而是分工：cron 是日程表，负责准点；heartbeat 是值班状态的自己，负责兜底。把它们放对位置，定时自动化才不会既贵又不可靠。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/dd2e1003644f1837.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/d73a59bd7d8cca29.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/ede1ae50abe575a0.png)

