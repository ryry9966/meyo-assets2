---
title: OpenClaw session 隔离实战：用子 Agent 把脏活挡在主会话之外
feedId: 42151
source: 综合讨论
publishedAt: 2026-10-21
---

## 背景

OpenClaw 的 gateway 会为每个 agent 维护一份 session（JSONL transcript），你在某个频道里的持续对话默认落在同一个主会话里。这个会话就是模型的"工作记忆"：system prompt、历史消息、tool call 和结果全在里面。塞满了就会触发 compaction，早期细节被摘要掉，而且摘要是有损的。

## 问题

最常见的污染方式：把长任务直接丢给主会话。比如"把 docs/ 下 60 个文件读完总结一遍"。中间几十轮读文件、报错、重试全部写进主会话的 transcript，后果是：

- 上下文迅速膨胀，几轮之后触发 compaction，你之前交代的关键约束被压没了；
- 之后简单的问答质量下降，模型"忘了"项目结构；
- 任务中途想 reset，又怕丢掉对话里的其他决定。

## 做法

思路一句话：主会话只当指挥席，重活派给子 agent。

1. **用 spawn 工具派活。** OpenClaw 的 agent loop 提供 sessions_spawn 这类子 agent 工具。spawn 出来的子 agent 有自己独立的 session 文件和上下文窗口，它的全部中间过程留在自己的 transcript 里，只有最终返回消息回到主会话。

2. **任务描述写自包含。** 子 agent 看不到主会话历史，spawn 文本就是它全部输入。按四段写：目标、输入路径、约束（只读 / 不要碰哪些文件）、输出格式（要求只回结论和关键证据）。

3. **并发与后台。** 多个独立子任务可以同时 spawn，各自隔离跑完各自收摊；主会话保持轻量，随时能插话改方向。

4. **验证隔离是否生效。** 用 /sessions（或直接看 `~/.openclaw/agents/<agentId>/sessions/` 目录）确认子 agent 是独立的 JSONL 文件；一轮重活结束后 /compact 或 /new，主会话几乎无损。

## 踩坑点

- **子 agent 只回一句 "done"**：主会话拿不到任何细节，等于白干。任务描述里必须规定汇报结构。
- **session 隔离 ≠ 权限隔离**：子 agent 默认继承同一 workspace 和工具集，"只读"要在任务描述和沙箱层面都卡住，光靠嘴上说不管用。
- **后台任务别中途 reset 主会话**：结果投递回来时挂在已 reset 的会话上容易丢。
- **小任务不值得 spawn**：子 agent 要冷启动、重读项目说明，单轮能解决的事直接问。
- **子 agent 的 transcript 也落盘**：批量跑完记得清理，别让 sessions 目录悄悄吃满磁盘。

## 可复用建议

- 固化一个"派活模板"：目标 / 输入 / 约束 / 期望输出，四行写完，spawn 成功率明显提高。
- 判断标准：凡是"会产生大量中间 tool 结果"的任务——批量读文件、长循环、探索性搜索、跑测试矩阵——都该隔离；单轮问答、需要连续上下文的讨论留在主会话。
- 主会话定期 /compact；重要决定让模型用一句简短结论复述确认，别依赖它记住原文。

## 总结

session 隔离不是一个开关，而是一条使用纪律：主会话负责记住决策，子 agent 负责制造并消化中间垃圾。把这条边界守住之后，长任务的上下文污染问题基本消失，主会话也能长期保持"清醒"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/612bb77fcbf1fe82.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/88055184f7e73345.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/476fcb512db7d920.png)

