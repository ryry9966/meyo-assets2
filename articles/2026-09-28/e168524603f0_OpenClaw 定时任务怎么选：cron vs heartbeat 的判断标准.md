---
title: OpenClaw 定时任务怎么选：cron vs heartbeat 的判断标准、配置与踩坑
feedId: 39281
source: 综合讨论
publishedAt: 2026-09-28
---

## 背景

OpenClaw 里有两套“定时”机制，很容易被混为一谈：

- **heartbeat**：主会话的周期心跳。网关按 `agents.defaults.heartbeat.every`（如 `30m`、`2h`）向主会话注入一条 HEARTBEAT 消息，Agent 对照 workspace 里的 `HEARTBEAT.md` 检查有没有事做，没事就回 `HEARTBEAT_OK`，不会转发到聊天渠道。
- **cron**：网关内置调度器，任务持久化在 `~/.openclaw/cron/jobs.json`，用 `openclaw cron add / list / runs` 管理。支持一次性（at）、固定间隔（every）、五段 cron 表达式，可指定时区；会话可以跑在 main，也可以跑在 isolated，结果能 announce 到 Telegram 等渠道。

两者都能“定时驱动 Agent”，但设计意图完全不同。

## 问题

实际使用中最常见的三种选错方式：

1. **把 heartbeat 当万能轮询**：interval 调到 5 分钟，让它去查 GitHub 通知、日志、天气。心跳每次都是一次完整的模型调用，没事干也在烧 token，主会话上下文还被无关结果污染。
2. **把需要“记忆”的活交给 cron**：job 的 prompt 里写“继续刚才那个问题”，但任务跑在 isolated session，上下文是空的，Agent 只能瞎猜。
3. **时区与运行前提没想清楚**：cron 相对网关时区计算；而且网关进程不跑，任务就不跑。

## 判断标准：一句话

**这个任务需不需要主会话的上下文？**

- 需要 → heartbeat。典型：整理当前会话待办、按清单例行巡检、检查配置漂移。
- 不需要、但要准点/独立产出 → cron。典型：早报推送、一次性提醒、定点抓 API 后投递到群。

## 做法

heartbeat 侧：

1. 在 `~/.openclaw/workspace/HEARTBEAT.md` 写清单，条目必须“可一眼判断有无事发生”，控制在个位数。
2. `agents.defaults.heartbeat.every` 从 `"2h"` 起步，需要限定时段再看 activeHours 类配置。
3. 观察返回 `HEARTBEAT_OK` 的比例：越高，说明间隔还可以拉长。

cron 侧（flag 以 `openclaw cron add --help` 为准）：

```bash
openclaw cron add --name morning-report \
  --cron "0 8 * * *" --tz Asia/Shanghai \
  --session isolated \
  --message "抓取昨日 commit 记录并总结成要点" \
  --channel telegram --deliver announce
```

先用 `openclaw cron run <jobId>` 手动触发验证 prompt 效果，再挂周期；用 `openclaw cron runs <jobId>` 查历史失败原因。

## 踩坑点

1. **HEARTBEAT.md 写成小作文**：条目一多，每次心跳都变成大任务，token 和延迟双爆炸。
2. **isolated job 依赖主会话记忆**：“继续昨天的事”在 isolated session 里无从谈起。要么把状态落盘到 workspace 文件让 job 自己读，要么改用 main session。
3. **心跳间隔过短**：空转也花钱。急事用 cron 的一次性任务补，别压心跳间隔。
4. **时区**：显式加 `--tz`。服务器在 UTC、人在东八区，早报会整整迟到 8 小时。
5. **拿 cron 当系统级服务**：任务状态在网关进程里，OpenClaw 停了任务就停，重启后的补跑行为也要按版本文档确认。

## 可复用建议

- 一句话模型：**心跳 = 常驻哨兵，回答“有没有要做的”；cron = 精确闹钟，回答“几点做什么”**。
- 主会话上下文很贵，定时任务别免费蹭：能落盘的状态一律落盘，让 isolated job 自取。
- prompt 没验证过就不要固化成周期任务，先用手动触发跑通再上线。

## 总结

heartbeat 和 cron 不是竞争关系，而是两个层次：心跳负责常驻感知，cron 负责定时执行。把巡检留给心跳、把精确调度交给 cron，token 成本和上下文质量都会好看很多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/56e7ec31e88e2811.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/87bf0f9cea80b786.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/746db830de67b4b0.png)

