---
title: OpenClaw 定时任务选型：cron 与 heartbeat 的分工边界
feedId: 37766
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

用 OpenClaw 做自动化，绕不开两个机制：**heartbeat（心跳）** 和 **cron**。两者都能让 agent 定期干活，但语义完全不同。我最初混用：巡检挂 cron、定点任务写进 HEARTBEAT.md，结果要么 agent 在没事时反复打扰群聊，要么定点任务时间不准还偶发漏跑。这篇梳理两者的边界，给一个可复用的选型思路。

## 问题：一句话区分

- **cron**：在确定的时间点，做确定的事。"每天 8:00 发日报"。
- **heartbeat**：每隔一段时间醒一次，自己判断有没有事。"每 45 分钟看一眼，没事别吭声"。

cron 是确定性调度；heartbeat 跑在主会话里，带对话上下文，由模型决定要不要回应（无事可回 `NO_REPLY`，频道侧静默）。

选错的典型症状：

- 巡检挂了 cron → 每次固定产出、固定推送，没有"要不要说"的判断，噪音大。
- 定点任务写了 HEARTBEAT.md → 依赖模型在心跳时"记得"去执行，时间不准，还可能漏。

## 做法

**1. cron 承接定点任务**（参数名以 `openclaw cron add --help` 为准）：

```bash
openclaw cron add \
  --name "morning-digest" \
  --cron "0 8 * * *" \
  --tz "Asia/Shanghai" \
  --session isolated \
  --message "汇总我 watch 的仓库昨天的 release，输出不超过 5 条"

openclaw cron list
openclaw cron run <job-id>   # 手动触发验证
```

要点：定点任务用 isolated 会话跑，不占主会话上下文；产出通过 delivery 推到目标频道。

**2. heartbeat 承接巡检任务**。工作区根目录的 `HEARTBEAT.md` 里写"检查项"，不是"任务"：

```markdown
- 日历 1 小时内有会议则提醒我
- ~/reports 出现新文件则摘要
- 其余情况一律不回复
```

间隔与小模型在配置里调（`heartbeat.every` / `heartbeat.model`）：

```jsonc
{
  "agents": {
    "defaults": {
      "heartbeat": { "every": "45m" }
    }
  }
}
```

**3. 组合形态**：cron 负责所有"重活"（抓取、汇总、定点推送），heartbeat 只留 3~4 条轻巡检（日程、异常文件、待办跟进）。

## 踩坑点

1. **heartbeat 配了主力模型**。一天几十次唤醒，成本可观。心跳应换小模型，只做"判断是否有事 + 简单动作"。
2. **HEARTBEAT.md 写成愿望清单**。超过 5 条后每次唤醒评估变慢、误报变多，宁少勿多。
3. **cron 忘配时区**。默认跟随网关时区（服务器多为 UTC），"早上 8 点"可能变成下午 4 点，任务级显式指定 tz。
4. **定时依赖网关存活**。两种机制都以网关进程活着为前提，笔记本休眠、服务器重启都会漏。任务有持久化，但补跑行为要自己验证：重启后 `cron list` 核对、翻日志。
5. **isolated 会话文件堆积**，需定期清理；delivery 全开会刷屏，按需开。
6. CLI 传长 message 的引号转义很烦，复杂 payload 建议用控制台编辑。

## 可复用建议

- 固定时刻 + 固定动作 → **cron**（isolated + 定向推送）。
- 周期巡检 + 是否打扰交给 agent 判断 → **heartbeat**（小模型 + 精简清单）。
- 分钟级高频轮询两者都不合适：用 cron 的短间隔 every，或干脆外部脚本打 Webhook。
- 需要对话上下文的轻提醒才用 heartbeat；不需要上下文的一律 cron isolated。
- 新任务先手动 `run` 验证，跑通再放开定时。

## 总结

cron 管"何时做"，heartbeat 管"要不要做"。把确定性动作从心跳清单里挪进 cron，把判断型巡检从 cron 里挪回 heartbeat，噪音和成本会同时下降。建议先跑通最小闭环——一个 cron 任务加一条心跳检查项——再逐步扩展。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/b96fb7623fd30fc0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/df9be454e9bd23bf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/e9625e82728ea8d4.png)

