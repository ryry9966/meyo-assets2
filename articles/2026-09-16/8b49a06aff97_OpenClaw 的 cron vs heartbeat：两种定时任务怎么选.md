---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 37734
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

OpenClaw 的 agent 常驻后端之后，"让它自己动起来"有两条官方路径：**cron** 和 **heartbeat**。两者都能让 agent 周期性醒来干活，但设计意图完全不同。常见的新手误区是把"每天 9 点发日报"写进 HEARTBEAT.md，或者用 cron 表达式硬凑巡检逻辑——结果不是任务漂移，就是 token 账单难看。

## 两者的本质区别

**cron：到点触发，动作确定。**
- 配置里声明 schedule（cron 表达式或 every 间隔）+ 一段 prompt
- 到点起会话执行，跑完即走
- 关键词：确定时间、确定动作、无需上下文

**heartbeat：周期脉冲，agent 自主判断。**
- agent 每 N 分钟（默认 30 分钟）醒来读一次 HEARTBEAT.md 的 checklist
- 自己判断这轮心跳是 SKIP 还是执行某条任务
- 关键词：模糊巡检、决策权在 agent、持续在线

## 做法

**cron 的标准姿势：**

```bash
openclaw cron add "日报" \
  --schedule "0 9 * * 1-5" \
  --session isolated \
  --prompt "读取 /data/metrics 昨日数据，生成三段式日报，发到通知频道"
```

三个要点：
1. 用 `--session isolated` 隔离会话，避免污染主对话上下文
2. prompt 必须自包含——isolated 会话没有历史记忆，路径、格式、接收方都写死
3. timezone 显式指定，别赌服务器时区

**heartbeat 的标准姿势：**

在 agent 工作目录维护一份 HEARTBEAT.md：

```markdown
# Heartbeat Checklist
- 检查 /data/queue 积压是否超过 50 条，超过则通知
- 检查磁盘占用是否超过 85%，超过则通知
- 其余情况一律回复 SKIP
```

巡检类任务间隔不建议低于 15 分钟。

## 踩坑点

1. **精确时间任务写进 HEARTBEAT.md**——"9 点发日报"在心跳里只能做到"9 点后的第一次心跳执行"，天然漂移。精确时间交给 cron。
2. **checklist 太长**——每轮心跳 agent 都会读一遍并思考，写 10 条基本每轮都"真干活"，成本直接乘上去。控制在 3~5 条，并写明 SKIP 条件。
3. **cron 用 main session**——定时任务的中间输出混进主对话，后续聊天 agent 会引用这些垃圾上下文。除非刻意要它记住，否则一律 isolated。
4. **isolated 会话里引用"上文"**——prompt 写"继续昨天的事"会直接失败，isolated 就是一张白纸。
5. **心跳任务太重**——单次心跳跑长任务会阻塞主循环，下一轮心跳被推迟。心跳只做轻判断+通知，重活留给 cron。

## 可复用建议

一个判断式就够了：

> **"几点几分必须做什么" → cron；"隔一阵子看看有没有事" → heartbeat。**

更进一步可以组合：cron 负责确定性动作（定时拉数据、落盘、发例行报告），heartbeat 负责异常巡检（读 cron 落下的数据，发现异常才发声）。两者叠加，agent 才像一个真正值班的服务，而不是一个被闹钟反复吵醒的人。

## 总结

| 维度 | cron | heartbeat |
|---|---|---|
| 触发方式 | 精确调度 | 周期脉冲 |
| 决策方 | 你（配置写死） | agent（读 checklist 判断） |
| 上下文 | isolated 独立会话 | 主会话在线判断 |
| 适合 | 定时报告、例行拉取 | 巡检、监控、异常发现 |
| 成本 | 按任务次数 | 心跳频率 × checklist 复杂度 |

先问自己：这个任务的触发条件能不能写成确定的时间？能，就用 cron；不能，再考虑 heartbeat。别让巡检逻辑承担调度职责，也别让调度系统承担判断职责。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/35b4a19991ac68e1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/67dd2dcc1d633d0c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/15aac102ee14ae15.png)

