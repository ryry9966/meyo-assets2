---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 41126
source: 综合讨论
publishedAt: 2026-10-10
---

## 背景

在 OpenClaw 里，一个 agent 在每个通道、每个对端上都有一个主 session（main session）。你在 Telegram/微信里聊的所有内容、触发的所有工具调用，默认都堆在这条会话的上下文里，靠 idle 时间或 reset 指令重置。

一旦开始做自动化——定时任务、批量处理、把大任务拆给子 agent——就会遇到本篇的主题：**子 agent 干活的痕迹，怎么不留在主会话里**。

## 问题：什么算"污染"

三种典型情况：

1. **过程回流**：子 agent 的中间工具调用、长篇输出被写进主会话历史，上下文膨胀，模型注意力被无关内容稀释，token 账单跟着涨。
2. **串台**：子 agent 或插件直接往主 session key 发消息，用户聊着 A，中途插进来一句 B 的进度汇报。
3. **长任务占坑**：后台任务复用主 session 跑批，对话中途上下文被冲掉，体感就是"它失忆了"。

排查方法很直接：跑 `openclaw sessions`，看主 session 对应的 `.jsonl` 是不是异常肥大，翻一翻里面有没有本不该出现的工具调用记录。

## 做法

1. **子任务一律 `sessions_spawn`，不要 `sessions_send` 进 main**。spawn 出来的子 agent 在 `~/.openclaw/agents/<agentId>/sessions/` 下有独立会话文件，结束后只把 summary 返回给调用方。
2. **spawn 的 prompt 要自包含**：任务目标 + 输入路径 + 输出格式 + 超时条件，四段式写清楚。不要把主会话历史整段贴进去——贴了就等于没隔离。
3. **长跑任务用独立 agentId**。cron、webhook 这类没人盯着的事，配一个专职 agent，和交互主会话物理分开。
4. **约定回传格式**。让子 agent 只回结论和关键数据（写进 results 文件或输出 JSON），主会话读结论，不读过程。
5. **定期巡检**。列一下体积 Top 的 session，超过阈值的归档或清理。

## 踩坑点

- **文件系统是共享的**。session 隔离 ≠ 工作区隔离：子 agent 改了工作区文件，影响同样会"回流"到主会话后续行为。需要硬隔离时，给子 agent 指定独立 cwd 或临时目录。
- **不设超时**。子 agent 卡在一个工具调用上，session 会一直占着，最好显式给 timeout。
- **同名 sessionPrefix 复用**。两个子 agent 用了相同前缀和名字，会话被复用互相覆盖，排查时容易误判成模型抽风。
- **图省事复用主 session 跑批**。插件里为了少写几行配置直接用 main key，短期没事，量一上来主会话必脏。

## 可复用建议

一条原则：**主会话只留对话与决策，子会话承担执行与噪音**。判断标准很简单——这段内容一个月后还需要出现在主上下文里吗？不需要，就 spawn 出去。

spawn prompt 模板可以直接固定下来：

```
目标：<一句话>
输入：<文件/数据路径>
输出：<格式约定，如 JSON schema>
约束：<超时、停止条件、禁止外发的范围>
```

再配一个几行的巡检脚本挂在 cron 上，每周报一次 session 体积，异常增长早发现。

## 总结

session 隔离不是某个配置项一开就完事，本质是上下文管理策略：**哪里产生噪音，就在哪里建边界**。做得好，主会话轻、响应稳、成本可控；做得不好，spawn 只是形式主义，上下文照样被稀释。建议从"cron 任务迁移到独立 agent + spawn prompt 模板化"这两件小事开始改，收益很快能看到。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/533bc1a03e113cd5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/a7fb6072e5f96c7f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/54b0339a7cc53fc0.png)

