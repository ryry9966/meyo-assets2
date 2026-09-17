---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 37958
source: 综合讨论
publishedAt: 2026-09-17
---

## 背景

OpenClaw 给 agent 提供了两套"定时干活"的机制。**cron** 由调度器按 cron 表达式（或一次性 at）到点触发，执行一条你预先写好的 prompt；**heartbeat** 则是让 agent 按固定间隔醒来，读一遍 `HEARTBEAT.md` 里的检查清单，自己判断"要不要做事"。两者都能做自动化，但触发方式、上下文和成本模型完全不同，选错了轻则浪费 token，重则 agent 半夜疯狂给你发消息。

## 核心差异

| 维度 | cron | heartbeat |
|---|---|---|
| 触发 | 表达式到点必跑 | 固定间隔唤醒，agent 可跳过 |
| 决策权 | 在你，payload 写死 | 在 agent，读清单判断 |
| 会话 | 可选 isolated / main | 固定跑在 main session |
| 成本 | 触发才消耗 | 每次心跳都消耗，哪怕结论是"没事" |
| 交付 | announce / deliver / none | 由 target 配置决定 |

## 怎么选：三步走

**第一步，按确定性给任务分类。** 有明确时间点的——每天 9 点汇总昨日会话、每周一提醒周报，归 cron；"隔段时间看一眼，没事别说话"的——收件箱巡检、磁盘水位检查、目录变更监控，归 heartbeat；外部事件驱动的不硬凑定时，走 webhook。

**第二步，配置 cron。** 大致是：

```bash
openclaw cron add \
  --name "每日会话纪要" \
  --cron "0 9 * * *" \
  --payload "汇总昨天的会话记录，提炼三条要点发给我" \
  --session isolated \
  --delivery announce
```

isolated session 每次都是全新上下文，payload 必须自包含，别指望它记得主会话里聊过什么。

**第三步，配置 heartbeat。** 设 `agents.defaults.heartbeat.every`（默认 30 分钟），把 `HEARTBEAT.md` 当值班手册写：

```markdown
- 检查 ~/inbox 是否有新文件，有则摘要通知
- 检查磁盘使用率是否超过 80%
- 若以上均无异常，直接跳过，不要回复
```

最后一句是重点：明确告诉 agent"没事就闭嘴"，否则它会为了证明没白醒而制造消息。

**第四步，短间隔验证。** 上线前把间隔调短（如 5 分钟）跑几轮，用 `openclaw cron list` 和运行日志确认触发时间、交付目标符合预期，再调回正常频率。

## 踩坑点

1. **时区。** cron 表达式按网关进程的系统时区解释，容器里默认 UTC，你写的"9 点"可能下午 5 点才响，启动时显式设 TZ。
2. **心跳频率失控。** 把 30m 改成 5m，一天 288 次心跳，每次至少一轮模型调用。先算账：频率 × 次数 × 单次成本。
3. **HEARTBEAT.md 写得太开放。** "看看有没有值得关注的"这种条目会让 agent 每次都能找到事汇报。条目要可判定、有明确阈值。
4. **一次性 at 任务不清理。** 跑完的 job 积在列表里，碍眼还可能被误触发。
5. **长心跳任务阻塞。** heartbeat 与消息回复共享 agent，巡检里塞个跑十分钟的活，用户消息就得排队。重活放 cron isolated。

## 可复用建议

- 把选型规则固化成一句话：**确定性找 cron，判断性找 heartbeat，事件驱动交给 webhook**。
- `HEARTBEAT.md` 进版本管理，像维护值班手册一样维护，每次 agent 误报就收紧一条措辞。
- cron 失败要可观测，把运行失败接进告警渠道，别等用户来问"今天的日报呢"。
- 避免职责重叠：同一件事既写了 cron 又放进 HEARTBEAT.md，只会双倍烧 token。

## 总结

cron 是你说了算的时刻表，精确、可隔离、成本可控；heartbeat 是 agent 的值班巡视，灵活但有判断噪音和持续开销。多数巡检类需求，一个 30 分钟的 heartbeat 加一份写得克制的 checklist 就够了；时间敏感的交付类任务交给 cron 更稳。先分类，再配置，最后用日志验证，三步走完，定时自动化基本不会翻车。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/71579b4afb0721d5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/bc13ed6acc695548.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/5b307dd61c8e18e7.png)

