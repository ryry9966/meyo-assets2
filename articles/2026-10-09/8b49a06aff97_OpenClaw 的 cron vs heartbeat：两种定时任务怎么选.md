---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 40970
source: 综合讨论
publishedAt: 2026-10-09
---

## 背景

Agent 用起来之后，第一个自然需求就是"让它自己动"。OpenClaw 给了两条路：**cron**（传统定时触发）和 **heartbeat**（周期心跳自检）。新手常见的两种翻车：用 heartbeat 做"每天 8 点发日报"，结果不准点还烧 token；用 cron 做"帮我盯着仓库"，结果条件一变任务就废了。

## 机制差异到底在哪

**cron**：你指定时间（cron 表达式或固定间隔），到点由 gateway 触发。prompt 在创建时就固定，默认跑在**隔离会话**里，跑完可以把结果 announce 到绑定的消息渠道。触发是确定性的——"几点干、干什么"都由你定。

**heartbeat**：Agent 每隔 N 分钟（默认 30m，openclaw.json 里配 `heartbeat.every`，设为 0 可关闭）自己醒一次，读工作区的 `HEARTBEAT.md`，结合**主会话上下文**自行判断要不要做事、要不要开口，可以判断完保持沉默。触发是代理性的——"要不要干"由模型判断。

核心差异就三点：**触发权**（你 vs 模型）、**上下文**（隔离 vs 主会话）、**成本**（按次 vs 每次心跳都是一次模型调用）。

## 三条判据

1. 有明确时刻或固定周期 → **cron**。如每天早报、每周一生成周报、每 2 小时拉构建状态。
2. 没有明确时刻，只有"条件满足才值得做" → **heartbeat**。如盯收件箱、看新 issue、电量提醒。
3. 混合场景：cron 做粗调度产出，heartbeat 做细巡视兜底。

## 具体做法

**cron：**

```bash
openclaw cron add --name daily-digest --cron "0 8 * * *" --prompt "..."
```

1. prompt 写成自包含的：隔离会话里它看不到你聊过什么，需要的背景要么写进 prompt，要么让它去读工作区文件。
2. 配好 delivery/announce 到 Telegram 等渠道，否则结果只躺在会话里没人看。
3. 定期 `openclaw cron list` 审计，清理僵尸任务。

**heartbeat：**

1. openclaw.json 里配 `heartbeat.every`（如 `"30m"`）。
2. 写 `HEARTBEAT.md`：短清单，每条写清"什么条件下做什么"，并显式写上"其余情况保持沉默"。
3. 观察几天日志，确认触发频率和噪音水平后再调间隔。

## 踩坑点

- **时区**：gateway 所在机器多半是 UTC，`0 8 * * *` 实际是北京时间 16 点。改机器 TZ 或在配置里显式指定时区，以你的版本支持为准。
- **心跳频率 = 成本**：每次心跳都是一次模型调用，哪怕它最后选择沉默。15 分钟一次一个月就是近三千次调用，建议心跳用便宜模型或放宽间隔。
- **HEARTBEAT.md 写太含糊**：要么频繁骚扰你，要么干脆什么都不报。统一用"仅当 X 发生才通知"的句式。
- **别用 heartbeat 实现准点动作**：心跳只保证"最晚多久会被看一眼"，不保证 9 点整发消息。
- **网关离线期间 cron 不触发**，重启后是否补跑要自己验证，重要任务别依赖补跑语义。
- **cron 需要记忆的场景**：让它先读工作区里的笔记文件，比往 prompt 里硬塞上下文稳。

## 可复用建议

- 一条决策句式：**能用 cron 表达式写清楚的，就用 cron；需要"看了才知道"的，才交给 heartbeat。**
- `HEARTBEAT.md` 控制在一屏内，只放当前真的在巡的事项，做完就删，别把它养成第二份记忆文件。
- 新任务先低频跑一周，确认产出有价值再加密频率；连续两周没触发过动作的任务直接删。
- cron prompt 里显式限定输出："只报告异常""不超过 100 字"，能有效降低 announce 噪音。

## 总结

cron 是"到点打卡"，heartbeat 是"定时巡逻"。定点、可预期的任务交给 cron；条件触发、需要模型判断的任务交给 heartbeat。两者不是二选一——**cron 产出 + heartbeat 兜底**才是更稳的组合。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/e226be384cbed8db.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/504dc8f617c28a92.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/e9cebf372ca0b7c4.png)

