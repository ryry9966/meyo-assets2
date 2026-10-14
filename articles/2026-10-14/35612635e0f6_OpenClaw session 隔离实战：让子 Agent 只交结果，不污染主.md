---
title: OpenClaw session 隔离实战：让子 Agent 只交结果，不污染主会话
feedId: 41543
source: 综合讨论
publishedAt: 2026-10-14
---

## 背景

OpenClaw 的主会话承载的是"决策层"上下文：用户意图、长期偏好、正在推进的任务状态。而很多活——全库检索、批量读日志、跑一轮验证——本质是脏活，会产生大量中间工具输出。我的常见做法是把这些活丢给子 Agent，但最早没管住隔离，踩了不少坑。

## 问题

症状主要有三个：

1. 子 Agent 的工具调用和结果直接写进主会话，几轮检索后 context 从 20k 涨到 100k+，compaction 提前触发，关键任务状态被压掉；
2. 子 Agent 读到主会话的无关历史，行为被带偏，比如把上一次调试的结论当成当前事实；
3. 并行跑两个子 Agent 时，互相看到对方写进同一个 session 的内容。

## 做法

现在的流程分四步：

1. **显式指定独立 session**。spawn 子 Agent 时不复用主 session id，给它独立标签（如 `sub-<task>-<date>`），确认 session 文件落在独立路径，而不是追加进主 session 的 JSONL。
2. **写最小化 brief**。子 Agent 的初始上下文只放三样：任务目标、可用工具白名单、返回格式约定。不要把主会话历史带进去——需要的事实由主会话显式摘出来写进 brief。
3. **约定返回契约**。子 Agent 最后只回一条结构化结论：结论、关键证据（文件路径/命令输出摘要，控制行数）、未尽事项。主会话里只落这一条，其余过程留在子 Agent 自己的 session 里，需要时按 session id 回查。
4. **收尾清理**。探索型临时 session 设定期限，跑完即归档或删除，避免本地 session 目录无限膨胀，也避免后续被错误复用。

## 踩坑点

- session 隔离只隔离对话上下文，**不隔离文件系统**。子 Agent 默认共享 workspace 和 git 工作区，并行改同一目录会互相踩，需在 brief 里约定只读范围或各自子目录。
- 返回契约不控长度，子 Agent 会把"摘要"写成另一次全量转述，等于污染换了个入口。给结论加行数上限。
- session id 复用是最隐蔽的坑：命名随意，下次同名字 spawn 会直接续上旧历史。
- compaction 对子 Agent 同样生效，超长任务里子 Agent 自己也会丢上下文，要主动拆段。

## 可复用建议

- 一任务一 session，命名带任务名和日期；
- brief 模板固定：目标 / 工具白名单 / 返回格式 / 只读范围；
- 返回契约固定：结论 + 证据路径 + 未尽事项，限长；
- 把"主会话只进结论，不进过程"写进 AGENTS.md，让模型自己长期遵守。

## 总结

session 隔离的核心不是"开个新会话"，而是控制上下文的流动方向：事实显式下行（brief），结果显式上行（契约），过程留在子会话。做完这套之后，主会话的 context 增长基本和任务数量线性相关，而不是和子 Agent 的工具调用次数相关——这才是隔离真正的收益。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/80307a0ffd5ff7ae.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/6b9bc2baeba769d8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/40ee7234ce84e0d9.png)

