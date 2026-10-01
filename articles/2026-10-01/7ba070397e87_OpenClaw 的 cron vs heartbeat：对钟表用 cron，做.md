---
title: OpenClaw 的 cron vs heartbeat：对钟表用 cron，做巡航用 heartbeat
feedId: 39968
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

OpenClaw 里有两套"定时"机制，新手很容易混淆：

- **heartbeat（心跳）**：默认每 30 分钟左右（带少量随机抖动）唤醒一次 agent，让它读工作区的 `HEARTBEAT.md`，顺手处理待办和巡检；没事就静默返回。
- **cron**：精确的计划任务，用 cron 表达式触发，每次跑一个隔离（或指定）会话，还能把结果投递到指定渠道。

两者都能"定期干活"，但设计目标完全不同。选错了不是不能用，而是会以很难排查的方式出问题。

## 问题

实际使用中常见的三类翻车：

1. 把"每天 9 点发日报"写进 `HEARTBEAT.md`，结果 agent 8:40 或 9:20 才轮到执行，时间漂移不可控；
2. 把重活塞进 heartbeat，主会话被占住，token 消耗翻倍；
3. cron 任务在容器里按 UTC 跑，配置的"早上 8 点"实际是下午 4 点。

## 机制差异与选型

一句话：**有明确时刻 → cron；"每隔一阵看一眼" → heartbeat。**

| 维度 | heartbeat | cron |
|---|---|---|
| 触发 | 固定间隔（约 30m） | 精确表达式/时刻 |
| 会话 | 共享主会话 | 默认隔离会话 |
| 任务来源 | 工作区 `HEARTBEAT.md` | 任务自身 payload |
| 输出 | 有事才说话，否则静默 | 可投递到指定渠道 |
| 适合 | 巡检、兜底、轻维护 | 日报、提醒、定点批处理 |

## 做法与步骤

**heartbeat：**
1. 工作区根目录建 `HEARTBEAT.md`，写 3~5 条以内、可勾选的巡检项，例如"检查 `~/data` 是否有超 1GB 的临时文件，超过才提醒"；
2. 间隔在配置里调整（`agents.defaults.heartbeat.every`，默认 30m，具体以你的版本文档为准）；
3. 明确写一句"无事不要回复"，让 agent 学会静默，这是控成本的关键。

**cron：**
1. 用 `openclaw cron add` 创建任务，写清名称、表达式、payload 和投递目标（参数以 `openclaw cron --help` 为准）：

```bash
openclaw cron add \
  --name "morning-brief" \
  --cron "30 8 * * *" \
  --message "汇总昨晚仓库动态，生成 5 条以内的晨报" \
  --deliver --channel telegram
```

2. 一次性提醒用 one-shot 模式（到期执行一次后自动清理），别常驻列表；
3. 每个任务设超时；创建前先跑一个"打印当前时间"的任务，验证容器时区。

**组合打法**：heartbeat 做廉价巡检，发现异常后，再升级为一次显式的 agent 调用或 cron 任务去处理。巡检便宜、处理精确，各干各的。

## 踩坑点

1. **heartbeat 不是调度器**：间隔是"大约"，agent 还可能判断无事可做直接跳过，别指望它准点；
2. **`HEARTBEAT.md` 越写越长 = 每半小时烧一次 token**，巡检项保持个位数，定期清退；
3. **cron 默认隔离会话**，看不到你的聊天历史，需要的上下文要写进 payload；
4. **时区**：容器里大概率是 UTC，第一个任务永远是验证时间；
5. 定点提醒不要放 heartbeat，用 one-shot cron，做完即走。

## 可复用建议

- 选型口诀：对钟表用 cron，做巡航用 heartbeat；
- 把 `HEARTBEAT.md` 当"值班的便利贴"，只放兜底类事项；
- cron 任务命名带时间语义（如 `morning-brief-0830`），排障时一眼可读；
- 重任务必须配超时和投递目标——失败要能被看见，而不是默默消失。

## 总结

cron 负责"什么时候必须做"，heartbeat 负责"闲时帮忙看着"。前者是日程表，后者是值班员。把定点、要投递、要留痕的事交给 cron；把巡检、兜底、轻提醒交给 heartbeat，两者的成本和可靠性才会都落在预期内。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/d8c05e8d0faa2285.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/2c408adf98062a88.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/7b55b750ff5f47c8.png)

