---
title: 不等你开口就把事办了：给 Agent 加一套可控的 Proactive 机制
feedId: 40873
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景

大多数 Agent 的默认形态是"应答机"：用户发消息，它回复，然后休眠。OpenClaw 生态里我们接了 MCP 工具、写了插件、配了自动化，但触发权始终握在用户手里。真正的效率增益出现在 Agent 具备 proactive 能力时——它自己发现问题、主动汇报，并在授权范围内把事处理掉。

## 问题

但"主动"在工程上很危险。朴素的实现（比如每五分钟跑一次 cron，让模型看看有什么要做的）会撞上三类故障：

1. **噪音**：没有增量信息也触发，用户被打扰几次后直接静音，通知通道作废；
2. **成本**：空转唤醒照样烧 token，5 分钟间隔的循环一个月能跑出可观的调用量；
3. **安全**：后台无人监督时执行写操作，出错没人兜底。

## 做法

实践中我们收敛出一套「触发 → 决策 → 分级执行」的三段结构：

**第一步，触发层。** 触发源分两类：定时类（cron，如每日日程摘要）和事件类（webhook / 文件监听，如 CI 失败、新邮件、日历变更）。关键原则：事件触发必须携带 payload，且在唤醒模型前先做一次脚本级增量检查——状态没变直接丢弃。

**第二步，决策层。** 唤醒后先通过 MCP 工具拉取上下文，再过一道明确的决策门：

```text
on event(payload):
    if not state_changed(payload.source): return   # 增量检查
    ctx = gather_context(payload, mcp_tools)       # 拉取上下文
    score = judge(ctx, user_goals)                 # 决策门
    if score < THRESHOLD: log_quietly(ctx); return
    if seen(dedup_key(ctx), window=24h): return    # 去重
    dispatch(tier=whitelist_tier(ctx), payload=ctx)
```

打分规则可以很简单：重要性 × 时效性 × 与用户目标的相关度，低于阈值转为静默记录。

**第三步，分级执行。** 所有主动行为分四档：

- **T0** 只写日志；
- **T1** 写入用户的每日笔记 / 待办；
- **T2** 推送通知给用户；
- **T3** 自动执行动作（仅限白名单，如"重试一次 flaky 测试"）。

新功能默认只开 T0/T1，跑两周、看过日志再逐级放开。

## 踩坑点

- **cron 时区**：调度器跑 UTC，用户在东八区，摘要凌晨五点推送。时间统一存 UTC，展示层转换。
- **payload 直进 prompt**：等于把 prompt injection 的口子开到后台。外部事件内容一律当不可信数据，先结构化提取，不拼原始文本。
- **静默失败**：后台任务挂了没人知道。必须有 dead-letter 日志和每日自检。
- **去重 key 粒度**：按"测试失败"这种粗分类会漏提醒，太细又重复打扰。按「资源 + 动作类型」粒度比较稳。

## 可复用建议

1. **先只读后写入**：第一版 proactive 只做汇总和建议，不动任何状态，验证置信度后再谈自动化。
2. **每次主动行为都是提案**：记录触发了什么、依据是什么、做了什么，这是后续调阈值和复盘的数据来源。
3. **加反馈回路**：用户可对主动消息标"有用/没用"，连续负反馈自动降频。
4. **预算硬顶**：给后台循环设每日 token 上限，超限降级为只记日志。

## 总结

Proactive 不是让 Agent 更"聪明"，而是给它加一套可控的调度与门控系统。核心就三件事：触发前有增量检查、执行前有决策门、所有动作可分级可回溯。先把打扰率和误报率压下来，主动能力才有人愿意一直开着。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/ac5a4516a6c0adf7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/9cc2d1624fc6375a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/995ed546d5eb1db2.png)

