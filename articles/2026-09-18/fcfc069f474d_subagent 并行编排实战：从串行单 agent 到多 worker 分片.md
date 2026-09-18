---
title: subagent 并行编排实战：从串行单 agent 到多 worker 分片
feedId: 38094
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

最近在 OpenClaw 里做两类活时撞到了单 agent 的天花板：一类是几十个模块的 API 迁移检查，另一类是多数据源的资料搜集汇总。共同特征是：任务可以切片，但单个上下文塞不下，串行跑又太慢。

单 agent 的问题不是“不够聪明”，而是三个工程问题：

- 中间产物（搜索结果、报错日志）把上下文撑爆，跑到后半段开始遗忘前面的约束；
- 串行执行，总时长等于各步骤之和；
- 中途一步失败，往往整轮重来。

## 做法

我的编排结构很简单：一个 orchestrator 加 N 个 subagent worker。

1. **按数据分片，不按步骤分。** 把 40 个模块切成 8 组，一个 worker 负责一组的完整流程。不要让一个 agent 只负责“读文件”、另一个只负责“改代码”——跨 agent 传递半成品状态是最容易出事的地方。
2. **orchestrator 只做三件事：** 切分、派发、汇总。它自己不碰具体执行，上下文保持干净。
3. **worker 用独立上下文加受限工具面。** subagent 配置里只挂必要的 MCP 工具，prompt 里写死输出格式（强制 JSON，带 `status / summary / artifacts` 字段）。
4. **并发与超时。** 信号量限到 3~4 个并发，单个 worker 超时 5 分钟，失败重试一次，再失败标记 skipped，不阻塞其他分片。
5. **结果落盘。** 每个 worker 把产物写到独立目录，orchestrator 只读汇总文件，不依赖 agent 的“复述”。

简化后的配置大致是：

```yaml
orchestrator:
  fanout: 4
  timeout: 300s
  retry: 1
subagent:
  tools: [fs.read, fs.write, mcp.linter]
  output: json         # status / summary / artifacts
  workspace: isolated  # 每人独立工作目录
```

实测一个串行约 40 分钟的任务，4 并发降到 12 分钟左右。代价是 token 消耗约 3 倍——每个 worker 都要重新加载任务说明和公共上下文，这笔账要提前算。

## 踩坑点

- **上下文继承太多。** 最初我把 orchestrator 的完整对话传给 worker，结果 8 个 worker 干了重复的活。改成只传“本组任务描述 + 公共约定”后，问题消失。
- **并发触发限流。** fanout 拉到 8 直接被 API 429。并发数不要一次拉满，先 3 个灰度再放大。
- **自然语言回传。** 早期让 worker“写段总结”，orchestrator 汇总时自由发挥出了不存在的结论。改成强制 JSON 结构后才稳定。
- **共享文件冲突。** 两个 worker 同时写一个索引文件互相覆盖。解法很土：各写各的分片文件，汇总阶段统一合并。
- **没有总超时。** 一个 worker 卡进死循环，整批任务陪着等。后来加了 per-task 和全局两级超时。

## 可复用建议

1. 先问一句：这任务真的需要并行吗？**能切片、子任务相互独立、单切片上下文可承受**，三个条件都满足再上编排，否则复杂度是负资产。
2. worker 数量从 3 开始，验证输出质量后再放大。
3. worker 任务模板化：同一段 prompt 只替换分片参数，方便 diff 和重放。
4. 幂等加断点续跑：产物落盘带状态标记，重跑只补失败分片。
5. 成本预算前置：并行编排的 token 账单近似按 worker 数线性增长，大批量开跑前先小规模试算。

## 总结

subagent 编排的本质不是“更多 AI”，而是**隔离与汇总**：每个 worker 有干净的上下文和受限的能力面，orchestrator 只负责切分与合并。编排逻辑写得越笨、越确定，整体越稳。花哨的动态任务规划收益很小，老实分片、结构化回传、超时重试，就能覆盖大部分并行化场景。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/2ca1a244e8ec8056.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/50ee4a859e9952ae.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/8f63d8aa5a8ea7fa.png)

