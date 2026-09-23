---
title: cron vs heartbeat：OpenClaw 定时任务选型的一次讲清
feedId: 38653
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

Agent 常驻之后，下一个自然需求就是"让它自己动起来"。OpenClaw 给了两种机制：

- **heartbeat**：固定间隔的心跳 tick，把一段轻量提示注入主会话，由模型自己判断"这轮要不要干活"；
- **cron**：标准 cron 表达式调度，到点拉起一次**独立会话**，执行你写死的 prompt。

新手最常见的错误是把所有定时需求都塞进 heartbeat，结果要么消息轰炸，要么整点任务永远不准时。这篇讲清两者的边界。

## 本质区别

| 维度 | cron | heartbeat |
|---|---|---|
| 触发逻辑 | 时间驱动，确定性 | 状态驱动，非确定 |
| 会话 | 每次全新隔离会话 | 共享主会话上下文 |
| 干什么 | 执行明确写好的 prompt | 模型自行判断 |
| 成本 | 只在触发时花钱 | 每次心跳都过一遍模型 |

选型前问自己三个问题：

1. 有精确时间要求吗？整点日报 → cron。
2. 触发条件能用一段 prompt 写清楚吗？能 → cron；只能靠模型"感觉" → heartbeat。
3. 需要最近对话上下文吗？需要 → heartbeat。

## 做法

cron 示例：

```bash
openclaw cron add --name daily-digest \
  --cron "0 8 * * *" \
  --prompt "汇总 ~/notes 昨日的改动，输出不超过 10 条摘要" \
  --deliver channel:telegram
```

heartbeat 示例（openclaw.json）：

```json
"heartbeat": { "every": "30m" }
```

心跳提示词保持一句话：「检查未处理事项，无事则返回 HEARTBEAT_OK」。模型回 HEARTBEAT_OK 时不触发投递，这是最低成本路径。

**组合模式（推荐）**：cron 负责重活（日报、备份、定时抓取），heartbeat 只做兜底巡检——"有排队未回的消息就处理"。

## 踩坑点

1. **心跳太密 + 主会话上下文大 = token 账单起飞**。心跳每次都携带完整 system prompt 和会话历史，间隔越短、会话越肥，成本越高。
2. **heartbeat 不保证精确时间**。它是"大约每 30 分钟"，宿主机休眠、网关重启都会丢 tick，且不补跑。想"每天 9:00 准点发"，必须 cron。
3. **cron 是隔离会话，没有记忆**。prompt 必须自包含：路径、口径、输出格式全部写死。写"按之前的约定"必然翻车。
4. **任务耗时超过间隔会重叠执行**。给重任务加锁（lockfile，或 prompt 里先检查上次产物的时间戳）。
5. **HEARTBEAT_OK 约定没写死时**，模型倾向于"说点什么"，凌晨三点收到"今天没有特别的事"就是这么来的。

## 可复用建议

- 默认原则：**能用 cron 就用 cron**，heartbeat 只做枚举之外的兜底。
- 心跳间隔 30 分钟起步，跑一周看 token 消耗再调。
- 把 cron prompt 当"交接文档"写：输入路径、处理规则、输出去向。
- 每月审计一次 cron jobs，清掉不再投递的僵尸任务。

## 总结

cron 是"到点干活"，heartbeat 是"定时醒来自己判断干不干"。前者买确定性，后者买自主性，两者的成本与可靠性差异都源于此。选型唯一标准：**触发条件是否可枚举**。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/d8ddd333d1744d11.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/1ec2f7ec145d0a8a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/ff40eefbddbf6190.png)

