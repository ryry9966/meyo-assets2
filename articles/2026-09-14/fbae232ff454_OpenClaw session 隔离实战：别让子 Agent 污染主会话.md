---
title: OpenClaw session 隔离实战：别让子 Agent 污染主会话
feedId: 37429
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

用 OpenClaw 跑稍微复杂点的自动化，主 Agent 几乎必然要派子 Agent：批量检索、长文档摘要、MCP 工具链调用、代码生成验证。子 Agent 有自己的 session（独立上下文和工具调用记录），但隔离做得不到位时，这些内容会以各种姿势回流到主会话。我们的长跑 automation 早期就吃过亏：跑一晚上，主会话 context 被中间噪声塞满，模型开始"记得"子任务的失败重试细节，后续决策明显被带偏。

## 问题：污染从哪来

拆下来主要是三个源头：

1. **结果回传污染**：子 Agent 把完整 transcript（含原始工具输出、失败重试）直接塞回主会话，一次子任务吃掉几千 token。
2. **状态泄漏**：子 Agent 和主 Agent 共享 workspace 或 memory，子任务顺手改了共享文件，主流程读到脏数据。
3. **生命周期失控**：子 Agent session 用完不清理，在常驻 automation 里越积越多，排查时根本分不清哪条记录是谁写的。

## 做法

现在团队内部模板的约定，可直接抄（字段名以你所用版本为准）：

```yaml
subagent:
  session:
    scope: isolated   # 独立 session，不继承主会话历史
    ttl: 30m          # 到期自动回收
  tools:
    allow: [search, http_get]  # 最小化白名单
  return:
    format: json_summary       # 只回结构化摘要
    max_tokens: 500
```

四条原则：

1. **独立 session + 显式 TTL**：子 Agent 用独立 session_id，主会话只保留一个"任务句柄"，用完自动回收。
2. **摘要契约回传**：子 Agent 结束只返回结构化 JSON（status / result_summary / artifacts 路径 / errors），原始产物写文件，主会话只拿路径。契约必须写进子 Agent 的 system prompt，不能靠自觉。
3. **工具白名单最小化**：检索型子 Agent 不给写权限，从根上砍掉状态泄漏。
4. **显式交接代替对话记忆**：跨 session 的数据统一走约定目录（如 `runs/{task_id}/...`），别指望模型"记住"。

## 踩坑点

- **TTL 设太短**：长任务被中途回收，子 Agent"失忆"，返回半截结果。按任务 P95 耗时加 buffer 来定。
- **契约没进 system prompt**：子 Agent 默认还是会把过程叙述一遍，摘要约束必须显式写死。
- **并发写同一目录**：多个子 Agent 互相覆盖 artifacts，按 task_id 隔目录能避掉九成这类问题。
- **排查困难**：子 Agent 的 session 日志要落盘保留，但和主会话日志分目录存，混在一起等于没存。

## 可复用建议

- 主会话只信"结论 + 指针"，不信过程。
- 任何跨 session 共享状态都当并发资源处理：有命名规范、有归属、可追溯。
- 隔离策略固化成模板，新任务从模板 fork，不靠每次手写。

## 总结

session 隔离本质上就两件事：**上下文不回流**（摘要契约 + token 预算），**状态不共享**（独立 workspace + 显式文件交接）。做到这两点后，我们的子 Agent 数量翻倍，主会话 context 增长基本持平，长跑 automation 的稳定性也明显改善。如果你们的场景更复杂——比如子 Agent 之间需要互相协作——欢迎评论区交流做法。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/ca2adc02b618fee6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/eaa663b6515aff3f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/513129568ff89456.png)

