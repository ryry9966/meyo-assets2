---
title: cron vs heartbeat：OpenClaw 两种定时唤醒机制的分工与选型
feedId: 43210
source: 综合讨论
publishedAt: 2026-11-02
---

## 背景

OpenClaw 里有两种让 agent 周期性干活的机制，新人很容易混用：

- **cron jobs**：由 gateway 内置调度器驱动，用 `openclaw cron add` 创建，支持 cron 表达式，每次触发跑一段独立的 prompt，默认使用隔离会话。
- **heartbeat**：主会话的周期性心跳，默认约 30 分钟一次，醒来读取工作区的 `HEARTBEAT.md`，逐条检查；无事则回复 HEARTBEAT_OK 静默。

常见翻车姿势有两种：把所有任务塞进 `HEARTBEAT.md`，主会话被轮询内容刷屏、token 成本失控；或者反过来全建 cron，结果所有 job 的 prompt 都要重复写一遍上下文。

## 怎么选：三个判断维度

| 维度 | cron | heartbeat |
|---|---|---|
| 时间精度 | cron 表达式，可对齐整点 | 固定间隔，随 gateway 启动时间漂移 |
| 会话上下文 | 隔离会话，干净但"失忆" | 共享主会话，记得之前聊过什么 |
| 成本模型 | 每个 job 独立 prompt，一次性 | 每次心跳都读整个 HEARTBEAT.md |
| 输出去向 | 显式投递到指定渠道 | 默认静默，有事才发言 |

我的经验口诀：**整点的事给 cron，惦记的事给 heartbeat**。需要固定时刻、精确间隔、无人值守可靠运行的，用 cron；需要"带着主会话记忆做轻量巡查"的，用 heartbeat。

## 具体做法

### 1. cron 做固定时间任务

```bash
openclaw cron add \
  --name "morning-digest" \
  --cron "30 8 * * *" \
  --message "读取 workspace/notes/todo.md，汇总今日待办，输出三段式摘要" \
  --session isolated \
  --timeout 120s
openclaw cron status   # 查看 job 列表和 last run
```

（具体 flag 以 `openclaw cron add --help` 为准。）关键点：隔离会话里 prompt 必须**自包含**——不要写"接着刚才的话题"，要写清楚读哪个文件、做什么、输出什么格式。

### 2. heartbeat 做环境巡查

`HEARTBEAT.md` 只放必须周期性判断的事项，控制在 3~5 条：

```markdown
# Heartbeat checklist
- 主会话中有未回复的用户提问时，提醒我
- tmp/reports/ 出现超过 1 小时未处理的新文件时，报告文件名
```

```json5
// openclaw.json
{ "agents": { "defaults": { "heartbeat": { "every": "45m" } } } }
```

无事时 agent 回 HEARTBEAT_OK，不产生任何输出，也不打扰主会话。

### 3. 混合模式（推荐）

cron 负责"准点发生"：日报 8:30 触发、每周清理任务。heartbeat 负责"随机抽查"：盯失败任务、看未回复消息。两者互补，不要二选一。

## 踩坑点

1. **HEARTBEAT.md 越写越长**。它相当于常驻 prompt，每次心跳都全文加载，成本线性上涨。凡是能定时的条目，全部迁去 cron。
2. **心跳污染主会话**。如果每轮都让它"总结一下近况"，主会话历史会混入大量心跳内容。坚持"无事静默"约定。
3. **cron job 失忆**。隔离会话看不到主会话，需要上下文就让 job 读工作区文件，或在 message 里显式带参数。
4. **时区问题**。cron 表达式按 gateway 所在机器时区解释，笔记本跨时区出差会翻车；heartbeat 间隔从 gateway 启动起算，不与整点对齐。
5. **休眠漏跑**。笔记本合盖期间 cron 会错过触发。时间敏感的任务，把 gateway 跑在常开的主机上，笔记本只做交互端。
6. **任务堆叠**。执行时长超过触发间隔时 cron 可能叠加，务必配 `--timeout`，并让任务幂等。

## 可复用建议

- 新任务先问一句：错过一次触发有影响吗？有影响 → cron 且跑在常开主机上；没影响 → heartbeat 或低频 cron。
- `HEARTBEAT.md` 当成付费库存管理：加一行之前先问能否删两行。
- 每月跑一次 `openclaw cron status`，看 last run 确认 job 真的在跑，删掉不再有人看的投递目标。
- 输出纪律：cron 一律显式投递到指定渠道；heartbeat 默认静默，确认需要打扰再开口。

## 总结

cron 和 heartbeat 不是竞争关系，而是分工：**cron 是日程表，heartbeat 是巡更**。把"必须准点发生"的事交给 cron，把"顺手看一眼"的事交给 heartbeat，成本、上下文和可靠性就都能兼顾。选型本身只有一行决策成本，但选错了会持续产生噪音和账单——值得在动手前多想十秒。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/c30472cb175695a9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/90946c0c2d79b178.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/71a8fc780ab9466c.png)

