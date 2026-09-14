---
title: 让 Agent 学会分身：subagent 并行编排的工程实践
feedId: 37620
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

单 Agent 串行干活的天花板很快就会摸到：让它审查十个模块、清理一批日志、给多个仓库补测试，都是"先读上下文、再调工具、再写结论"的循环。循环一长，主上下文被中间过程塞满，模型开始遗忘最早的任务约束，速度也上不去。

subagent 编排的思路很朴素：主 Agent 只做拆解和汇总，把相互独立的子任务分发给多个拥有独立上下文的 subagent 并行执行。OpenClaw 的 agent 配置和 MCP 工具体系恰好提供了做这件事的基础件。

## 问题

直接开跑会遇到三件事：

1. **上下文污染**——子任务的工具调用明细如果全量回传，主 Agent 的上下文照样爆；
2. **写冲突**——两个 subagent 同时改同一个文件，后写的覆盖先写的；
3. **静默失败**——某个子任务失败了，但 subagent 返回一句"已完成"，主 Agent 毫无察觉。

## 做法

我们的做法分四步：

**第一步：判断任务可并行性。** 只有满足"输入独立、写入目标不重叠、产出可结构化描述"的任务才拆。重构 A 模块和给 B 模块补测试可以并行；同一个文件里改接口和改调用方不行。

**第二步：给每个 subagent 收窄权限。** 通过 MCP 工具白名单限制它能碰的资源，写权限精确到目录级：

```yaml
subagents:
  - name: refactor-mod-a
    tools: [fs:read, fs:write:src/mod-a, mcp:eslint]
    task: "重构 src/mod-a，保持对外接口不变"
    output: { format: markdown, max_tokens: 400 }
  - name: test-mod-b
    tools: [fs:read, fs:write:tests/mod-b, mcp:pytest]
    task: "为 src/mod-b 补齐单测"
    output: { format: markdown, max_tokens: 400 }
concurrency: 3
```

**第三步：约定输出契约。** 强制每个 subagent 返回固定结构：状态（done / failed / partial）、改动清单、遗留问题。`max_tokens` 卡住回传体积，主 Agent 只看摘要不看过程。

**第四步：主 Agent 做聚合与重试。** 收齐结果后逐条校验状态，`failed` 的子任务带着失败原因重新派发一次；仍然失败就标记为人工介入，不无限重试。

## 踩坑点

- **并行数不是越多越好。** 我们从 6 并发降到 3，反而总耗时更短——API 限流和 token 消耗都是隐性成本，排队比撞限流重试便宜。
- **写入分区必须显式声明。** 靠"我觉得它们不会冲突"是大忌，目录级 write 白名单要写进配置，而不是写在 prompt 里求模型自觉。
- **警惕 subagent 说谎。** 模型倾向报告成功。聚合阶段要求附上关键证据（测试通过数、diff 摘要），没有证据的 done 按 partial 处理。
- **重试要幂等。** 子任务开始前先让 subagent 检查目标状态，避免重试时在半成品上叠加改动。

## 可复用建议

整理成一张清单，任何想上 subagent 的场景先过一遍：

- 任务间是否真的无共享写入目标？
- 每个 subagent 的工具白名单是否最小化？
- 输出是否有结构化契约和长度上限？
- 并发数是否配了上限和超时？
- 失败路径是否有重试预算和人工兜底？

先拿两个并行的低风险任务（比如批量生成文档、批量跑 lint 修复）验证整条链路，再逐步放开到写代码类任务。

## 总结

subagent 编排解决的不是"模型不够聪明"，而是"一个上下文装不下十件事"。它的收益来自隔离与并行，风险也来自隔离——主 Agent 看不见细节，所以契约、白名单、证据校验这三件事必须做成机制，而不是口头约定。把编排层当工程做，多 Agent 才是真生产力；当玄学用，就只是多花三倍 token 排一个更长的队。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/d08178f7dcff2520.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/e81d3c1a4f68ea2f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/c0c2b7a1f7f4c1c5.png)

