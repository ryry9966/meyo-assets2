---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 37397
source: 综合讨论
publishedAt: 2026-09-13
---

## 背景

OpenClaw 的主会话（channels 里那个常驻对话）承载着用户的长期上下文，是整个系统里最“贵”的资源。跑自动化任务时，经常需要派生子 Agent 去干脏活：批量调 API、遍历一堆文件、跑长脚本。如果子 Agent 的工具调用和中间输出全部回流主会话，上下文很快被撑爆，模型注意力被垃圾信息稀释，主对话质量明显下降。

## 问题

不做隔离时的典型症状：

- 主会话历史里塞满子任务的工具调用记录，几轮之后关键指令被“淹没”；
- 子 Agent 重试三次的报错日志原样进了主会话；
- 子 Agent 直接改了 workspace 里的共享文件或记忆，主会话后续行为被悄悄带偏；
- 多个并行子任务读写同一个 session 文件，出现互相覆盖的竞态。

## 做法

1. **派生即隔离**。用 subagent 派生时显式指定独立 session（独立 session key），子任务的所有工具调用、中间输出落在自己的 transcript 里，主会话上下文不再被动增长。
2. **收口为结构化结果**。约定子 Agent 只返回一个紧凑的摘要或 JSON（结论、关键数据、失败原因），主会话只注入这一条，禁止把原始日志当结果回传。
3. **工具白名单收敛**。子 Agent 只给完成任务所需的最小工具集，避免它“顺手”调用会写主 workspace 的工具。
4. **生命周期管理**。给子 session 设超时，任务结束后主动清理；定期用 sessions 列表检查有没有孤儿 session。
5. **写路径隔离**。子 Agent 需要产出文件时，落到独立目录（如 `workspace/subtasks/<task-id>/`），主会话按需取用，不共享默认写入位置。

## 踩坑点

- 隔离了 session 但没隔离 workspace：子 Agent 改了共享的记忆文件，主会话照样被污染。
- 结果摘要太长，等于变相把日志搬回来，隔离白做。给结果设字数和条目上限。
- 子 Agent 再派子 Agent 没限制深度，任务树失控。嵌套通常 1~2 层足够。
- 并行子任务复用同一个 session key，transcript 互相覆盖。
- 清理逻辑只挂在成功路径上，失败和超时路径漏了，磁盘上堆积僵尸 session。

## 可复用建议

- 把子 Agent 的“返回契约”固化成 schema：`status / result / artifacts / errors` 四个字段，所有派生处复用同一套。
- 主会话注入结果时附一行任务元信息（任务 ID、耗时、工具调用次数），方便事后审计但几乎不占上下文。
- 把隔离配置沉淀成团队模板：一次配置，多处 spawn，避免每个任务各写一套。

## 总结

session 隔离的本质是控制上下文的**写入权限**：主会话只该看到“结论”，不该看到“过程”。派生时隔离、收口时压缩、退出时清理，三步做完，主会话就能长期保持干净、可控、可审计。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-13/ab309ee0444b31b3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-13/5158289ff8793f00.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-13/20d4cf55c254a4db.png)

