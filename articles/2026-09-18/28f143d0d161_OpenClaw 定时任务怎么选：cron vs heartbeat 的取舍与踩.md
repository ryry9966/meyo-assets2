---
title: OpenClaw 定时任务怎么选：cron vs heartbeat 的取舍与踩坑
feedId: 38023
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

OpenClaw 里的常驻 agent 不只是被动应答，它有两种"自己醒来"的机制：

- **heartbeat（心跳）**：主会话内的周期性自省。每隔一段固定时间给主会话注入一次心跳事件，agent 自己判断要不要做事；没事就回一个静默的 `HEARTBEAT_OK`，用户无感。
- **cron（定时任务）**：旁路调度。用 cron 表达式定义，到点后以**隔离会话**跑一段固定的 prompt，结果可投递到指定渠道，也可以不留痕。

两者都叫"定时"，但运行模型完全不同。选错会出现 token 浪费、主会话上下文被撑爆，或者"任务定时跑了但没人看见结果"。

## 核心判断

就看两个维度：**是否依赖主会话上下文**、**时间精度要求**。

- 需要聊天记忆、与当前对话状态相关的轻量检查 → heartbeat。比如"如果用户昨天提过要跟进某事且还没办，就提醒一句"。
- 时间点明确、内容自包含、不需要聊天历史的任务 → cron。比如每天早上的消息摘要、每周清理临时文件、固定时间推日报。

一句话：heartbeat 是主会话的"自省"，cron 是旁路的"日程表"。

## 做法

heartbeat 配置（`~/.openclaw/openclaw.json`）：

```json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "45m",
        "activeHours": { "start": "08:00", "end": "23:00" }
      }
    }
  }
}
```

同时在系统提示里写死规则：心跳时无事可做必须回 `HEARTBEAT_OK`，不要硬找事做。

cron 任务（参数名以 `openclaw cron add --help` 实际输出为准，不同版本略有差异）：

```bash
openclaw cron add \
  --name "morning-digest" \
  --cron "30 8 * * 1-5" \
  --message "汇总过去24小时未读消息要点，生成不超过10行的摘要" \
  --deliver --channel telegram
```

三个要点：prompt 自包含（不指望它记得主会话）、显式指定投递渠道、约束输出长度。

## 踩坑点

1. **心跳间隔太短**。主会话上下文会被心跳记录持续撑大，token 成本线性上涨。建议 30–60 分钟起步，深夜用 `activeHours` 直接关掉。
2. **心跳"戏太多"**。不在系统提示里约束的话，agent 会为了响应心跳硬发消息打扰用户。务必写明"无事则静默"。
3. **cron 时区**。容器里 `TZ` 是 UTC，你以为是早八，实际下午四点才跑。给容器显式设 `TZ`，或手动换算表达式。
4. **cron prompt 带上下文假设**。隔离会话里没有聊天历史，"接着刚才说的"这种写法必然翻车，每次都要把背景写全。
5. **长任务重叠**。cron 到点就起新会话，上一轮没跑完也会再开一个。长任务自己做幂等/去重，或干脆拉长间隔。

## 可复用建议

- 先问一句：这个任务需要"知道之前聊了什么"吗？需要 → heartbeat；不需要 → cron。实测 90% 的定时需求属于后者。
- 输出物尽量收敛：摘要、清单、一句提醒，而不是长篇报告，否则投递渠道会变成垃圾场。
- 一次性任务用 cron 的一次性调度，别让它长期占着心跳。
- 上线前手动触发一次，确认投递目标和时区都对，再放它进生产。

## 总结

heartbeat 和 cron 不是竞争关系，而是两种运行模型：一个有状态、低精度、贴着主会话；一个无状态、高精度、隔离干净。把"自省"交给 heartbeat，把"日程"交给 cron，定时任务这条线基本就稳了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/60aa02bb80ee9394.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/fdb9a18c45e00bc6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/416f34e991eb2f55.png)

