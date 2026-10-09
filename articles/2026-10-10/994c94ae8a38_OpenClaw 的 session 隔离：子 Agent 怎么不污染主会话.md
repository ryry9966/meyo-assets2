---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 41058
source: 综合讨论
publishedAt: 2026-10-10
---

## 背景

主会话承担用户对话、长期记忆和上下文压缩，是整个系统里最不该被弄脏的地方。但实际跑自动化时，主 Agent 经常要委派任务——批量检索、代码执行、串一条 MCP 工具链——这时通常会起子 Agent。OpenClaw 里子 Agent 默认可以共享主会话上下文，调试时很方便，放进生产流程就是隐患。

## 问题

子 Agent 和主会话跑在同一个 session 里，污染主要来自三处：

1. **上下文膨胀**：子 Agent 的工具输出、中间推理全部追加进主历史，几轮下来 token 成本和 compaction 频率肉眼可见地上涨。
2. **记忆污染**：主会话做摘要压缩时，子 Agent 的临时噪音会被烘进 summary，之后对话方向被带偏，且很难排查原因。
3. **副作用泄漏**：子 Agent 如果拿到主会话全套工具，可能误触发有副作用的调用——发通知、改配置、覆写主工作区文件。

## 做法

我们团队现在的约定是：**子 Agent 会话是易耗品**，create → run → extract → destroy，一个任务一个 session。具体步骤：

1. **隔离 session**：spawn 时给子 Agent 独立 session id，不继承主历史，只接收按值传递的任务简报。
2. **收窄工具面**：子 Agent 只挂载本次任务需要的 MCP 工具白名单，不透传主会话工具集。
3. **工作区隔离**：给子 Agent 单独的 scratch 目录，产物落盘后主会话只拿路径或摘要。
4. **定义返回契约**：固定 JSON schema（`status` / `summary` / `artifact_paths` / `error`），主会话只读这四项。
5. **收口再返回**：要求子 Agent 结束前把发现压缩成有大小上限的 result，禁止原样回吐日志。
6. **生命周期管理**：设超时和显式 cancel，结束后归档或销毁 session，主会话只留一条委派记录（task id、状态、结果路径）。

```
主会话 ──task brief(JSON)──▶ 子Agent（独立session / 工具白名单 / scratch目录）
       ◀──result(JSON, ≤2KB)──
```

## 踩坑点

- **任务简报里塞整段主历史**：隔离形同虚设。简报只给目标、约束和必要背景。
- **忘了收工具**：子 Agent 顺手调了主会话的通知工具，给用户发了条测试消息。工具白名单必须在 spawn 前确定，不能事后补救。
- **异步结果撞上 compaction**：子 Agent 跑得慢，结果回来时主会话已经压缩过，引用丢失。长任务让子 Agent 把结果写文件，主会话轮询拿路径。
- **结果不压缩**：子 Agent 回吐 20KB 原始输出，主上下文照样被撑爆。契约里限制 result 体积，超限只给摘要。
- **session 复用**：上一次任务的残留上下文影响下一次判断，排查起来非常费时。坚持一次任务一个 session。

## 可复用建议

- **契约先行**：先写返回 schema，再写委派 prompt，两边都不会跑偏。
- 主会话维护一张轻量委派登记表，上下文保持很小，但每个任务可追溯。
- 跨界的东西越少越好：一次任务跨界的数据，理想情况下只有 task brief 和 result 两个 JSON。
- 定期审计子 Agent 的工具调用日志，发现越界调用就收权，而不是靠 prompt 里写"请不要"。

## 总结

Session 隔离的本质不是"分开跑"，而是**控制什么能跨过边界**：过程留在子会话里自生自灭，只有小体积的结构化结果回流主会话。做到这一点，主会话的记忆干净、成本可控，子 Agent 也才敢放手去跑长任务。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/dca3687cdd490c2e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/023cfd6feea0f89a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/2168029542855f29.png)

