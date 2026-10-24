---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 42503
source: 综合讨论
publishedAt: 2026-10-25
---

## 背景

跑自动化任务时经常需要拆分子任务：批量抓网页、跑数据整理、并行调 MCP 工具。这些活在 OpenClaw 里一般交给子 agent。但不少人跑完发现主会话变"脏"了——上下文被中间过程塞满，模型开始忘最初的指令，连回答风格都变了。这通常不是模型的问题，而是 session 边界没划清。

## 问题出在哪

主会话的上下文是共享的"工作记忆"。如果子 agent 的工具输出、报错、中间推理直接写回主会话历史，会有三个后果：

1. 无关 token 挤占窗口，长对话后主任务质量下降；
2. 子任务失败的重试记录留在历史里，模型后续倾向于重复错误路径；
3. 主会话 compaction 的摘要被噪声带偏，越压越失真。

## 做法

OpenClaw 的 session 隔离核心是：每个子 agent 有自己的 session 存储，父会话只收结果。实操分四步：

1. **用 sessions_spawn 派生，不要手动切会话。** 通过工具派生的子 agent 会拿到独立 session，跑完后只把最终摘要作为一条 tool result 写回父会话。手动 `/new` 再切回来的做法不可靠，容易把历史带进去。
2. **给子 agent 写"任务简报"，而不是转述全历史。** task 描述只放三样：目标、可用工具、输出格式约束。不要把主会话几轮对话原文粘进去——那是另一路污染源。
3. **约束返回体。** 要求子 agent 返回固定结构的摘要（结论、关键数据、产出文件路径）。返回体越小，父会话上下文越干净。
4. **跑完核对 session 文件。** 去对应 agent 目录确认子任务记录落在子 session 里，父会话只有一条结果记录。有偏差就查 spawn 方式。

## 踩坑点

- **session 隔离 ≠ 文件系统隔离。** 子 agent 默认共享 workspace，会写文件、改配置。要么给子任务指定独立输出子目录，要么在简报里写明"只读 / 只写 XX"。这是最常被忽略的一层污染。
- **别让子 agent 写长期 memory。** 子任务的中间结论一旦进 memory，会污染之后所有会话的召回。memory 写入留给主会话自己判断。
- **子 agent 自己也会 compaction。** 跑长任务的子 agent 同样会压缩上下文，工具输出量照样要控制，不是隔离开了就随便造。
- **并发派生注意 session key。** 复用同一个 key 可能命中旧会话的残留上下文，每个子任务用独立命名。

## 可复用建议

把"任务简报 + 输出 schema + 独立输出目录"沉淀成项目内固定模板，每次 spawn 填空即可。团队统一模板，比各自摸索省大量排查时间。自动化脚本里建议加一步：任务结束后 diff workspace，出现计划外写入直接告警——文件层的污染用脚本兜底，比靠模型自觉稳得多。

## 总结

session 隔离的本质是控制信息流向：下放任务时只给必要上下文，回收时只要结构化结果。OpenClaw 已经把 session 边界做进了派生机制，剩下的功夫在调用习惯上——简报写得越克制，主会话活得越久。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/00d51ddb800c027c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/a35354bfba44245b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/d73f896ef3adf361.png)

