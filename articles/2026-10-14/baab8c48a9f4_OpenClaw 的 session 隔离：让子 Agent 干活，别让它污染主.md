---
title: OpenClaw 的 session 隔离：让子 Agent 干活，别让它污染主会话
feedId: 41554
source: 综合讨论
publishedAt: 2026-10-14
---

## 背景

OpenClaw 的主会话按 agent + 通道 + 对话对象 维护，是长期累积的上下文。日常使用中，工具输出、文件内容、历史消息不断往里堆。上下文窗口是有限预算：塞得越多，模型对早期指令的遵循越差，成本也越高。

OpenClaw 提供了子 agent 机制：主会话通过 `sessions_spawn` 派生子 agent，子 agent 拿到独立 session（独立的 JSONL 会话文件），任务完成后只把最终结果回传，过程明细不进入主上下文。

## 问题

没有隔离时的典型“污染”：

1. 重型工具输出（长 grep、大文件读取、网页抓取）直接刷进主会话，几轮之后模型开始“忘事”。
2. 并行任务的中间状态交错在历史里，分不清哪条消息属于哪个任务。
3. 想回放某个任务的完整过程时，它和日常聊天混在一起，无法单独清理。

## 做法

以“子 agent 扫描仓库并输出依赖风险摘要”为例：

1. 主会话调用 `sessions_spawn`，task 写清五件事：目标、输入路径、输出路径、约束、返回格式。明确要求“只返回结论 + 关键文件路径，不超过 20 行”。
2. spawn 时指定独立 `cwd`，把文件操作限定在子目录，避免与主 workspace 冲突。
3. 给任务起 `label`，之后用 `sessions_status` 查进度，需要补指令用 `sessions_send`。
4. 子 agent 结束后，主会话只收到一条结果消息；过程日志留在子 session 里，到期归档。

主会话此时新增的上下文是几十行结论，而不是几千行扫描日志——这就是隔离的价值。

## 踩坑点

- **让子 agent 直接往主会话发消息**。子 agent 里再用发消息工具插话主会话，等于把污染带回来。约定：只通过结果回传，不额外发消息。
- **把大段内容塞进 task prompt**。子 agent 不会自动继承主会话上文，需要的信息要显式写进 task；但贴大文件原文会让它自己爆上下文，正确做法是给路径让它自己读。
- **共用 cwd 跑 git 或写同名文件**。两个子 agent 在同一目录操作会互相踩，并行任务各自指定工作目录。
- **并发数拉满**。`agents.defaults.subagents` 下的 `maxConcurrent` 不要按上限配，工具调用量大的任务 2–4 个并发就可能触发供应商限流。
- **归档后才想起看过程**。子 session 完成后一段时间会归档，需要审计的就调大 `archiveTimeoutMinutes`，或提前导出 session 文件。

（以上字段名以你所用版本文档为准。）

## 可复用建议

- **职责分层**：主会话只做调度与对话，重 IO、重搜索、批处理全部下沉到子 agent。
- **固化 task 模板**：目标 / 输入 / 输出 / 约束 / 返回格式五段式，减少每次现编的歧义。
- **控制回传体积**：结论 + 路径，禁止子 agent 把日志原文贴回来。
- **主会话定期 `/new` 重置**：配合隔离机制，长期运行的 assistant 主上下文才能一直保持轻。

## 总结

session 隔离的本质是上下文预算的分配：过程性、可丢弃的上下文放进短命 session；结论性、需要长期记住的内容才回主会话。用好 `sessions_spawn` + 独立 cwd + 受控回传这三件事，主会话就能在长期运行中保持稳定的质量。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/da062239e560a538.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/78ac39362a60bd55.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/5e3bbd58584e7530.png)

