---
title: 不等你开口：给 AI 助手装上可控的 proactive 触发链路
feedId: 39556
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

多数 Agent 的交互模型是"你问它答"：一条消息进来，跑一轮工具调用，返回结果。但真正省时间的场景往往是反过来的——每天早上的摘要、CI 挂掉的即时通报、截止日期临近的提醒。这类需求不需要人发起，需要的是触发器（trigger）驱动。OpenClaw 的 cron 任务和 webhook 入口给了做 proactive 的底座，难点不在"能不能主动"，而在"怎么主动得可控"。

## 问题

直接把"定时唤醒 + LLM"接上消息通道，跑两天通常会遇到三件事：

1. **通知疲劳**：Agent 觉得什么都重要，每小时推一条，用户第一反应是关掉它；
2. **成本失控**：每次心跳都跑完整推理，token 消耗是被动模式的几倍；
3. **自激循环**：Agent 自己发出的消息被当成新事件处理，开始给自己回话。

## 做法：watch → judge → act 三层

我们把 proactive 拆成三层，各层职责严格分开。

**第一层 watch：便宜的触发与预过滤**

- cron 定义固定任务（早报、周报）；
- webhook / 文件监听定义事件型触发（构建失败、新 issue、日历临近）；
- 预过滤用纯规则完成：关键词、来源白名单、静默时段。大部分噪音在这里被挡掉，不花一个 token。

**第二层 judge：LLM 只做"值不值得打扰"的判断**

- 通过预过滤的事件才进入模型，输出固定为三选一：`ignore / notify / act`，必须附带理由和证据引用；
- 用内容 hash + 时间窗做去重，防止同一事件反复触发；
- 每类触发器配独立预算：每天最多 N 条 proactive 消息。

**第三层 act：分级执行**

- `notify`：只推送结论 + 原因 + 证据链接，不执行写操作；
- `act`：仅限 allowlist 内的低风险动作（打标签、建草稿），高风险动作回推确认卡片；
- 所有动作带 idempotency key，重试不重复执行。

最小配置示意（字段按你的版本调整）：

```yaml
proactive:
  triggers:
    - type: cron
      schedule: "0 8 * * *"
      task: daily_brief
    - type: webhook
      path: /ci-failure
  judge:
    dedup_window: 6h
    daily_budget: 10
  act:
    allowlist: [draft_issue, add_label]
    require_confirm_above: write
```

## 踩坑点

- **自激循环最隐蔽**：务必给 Agent 自己发出的消息打标记，在 watch 层直接排除。不要指望 LLM 判断"这是不是我自己"。
- **静默时段看时区**：配置错一次，凌晨三点的"早报"会让用户永久关掉这个功能。
- **judge 层别自由发挥**：输出用固定枚举 + 理由字段，否则没法统计误报率、没法迭代。
- **先 dry-run 一周**：只记录"本来会发什么"，不真发。一周后人工审日志再放开，比上线就全开省很多信任成本。

## 可复用建议

1. **规则前置、LLM 居中、动作兜底**：能用 if-else 挡掉的绝不进模型。
2. 每条 proactive 消息必须自带 "why now"，用户一眼能判断该不该被打扰。
3. 把用户反馈（"这类别推了"）固化成 watch 层规则，而不是靠 judge 层的模糊记忆。
4. 预算、静默时段、确认门槛是三根安全绳，缺一不可。

## 总结

Proactive 不是"更聪明的 Agent"，而是一个工程问题：触发、过滤、预算、权限四个环节各司其职。先用 dry-run 收集真实数据，再逐步放开执行权限，主动助手才不会变成主动骚扰。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/59d5dd0e390e73f9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/dd062e548b005d4f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/761a7e8898096aaa.png)

