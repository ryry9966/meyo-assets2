---
title: 别让 Agent 排队干活：subagent 并行编排的工程实践
feedId: 37516
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

用单 Agent 干复杂活的人大概都遇到过同样的困境：一个 Agent 既做规划、又做检索、还要写文件，context 越滚越大，跑到后面开始"忘事"，而且所有步骤串行执行，五个独立子任务排队跑，时间线性增长。

subagent 模式是现在比较成熟的解法：主 Agent 只负责拆解任务和汇总结果，具体执行交给多个拥有独立 context 的 subagent 并行处理。我们在 OpenClaw 的自动化流程里跑了一段时间，把实践记录下来。

## 问题

单 Agent 模式的三个核心痛点：

1. **context 污染**：检索类任务的中间噪声挤占后续推理空间，长任务后半段质量明显下降；
2. **串行瓶颈**：互不依赖的任务也被迫排队；
3. **职责混杂**：一个 Agent 挂十几个工具，工具选择错误率随工具数上升。

## 做法

我们的编排结构分五步：

**1. 拆任务。** 主 Agent 先判断子任务是否真正独立：读写资源不冲突、无数据依赖。拿不准的宁可串行。

**2. 定义 subagent。** 每个 subagent 三要素：独立 context、受限工具集（最小权限）、明确的输入输出契约。比如"文档摘要 agent"只给读取工具，返回固定 JSON 结构。

**3. 并行派发。** 异步并发执行，并发上限从 3 起步，稳定后再提。不追求一次性拉满。

**4. 结构化回传。** subagent 只返回摘要 + 产物落盘路径，绝不全文回传。这是控制主 Agent context 的关键。

**5. 汇总校验。** 主 Agent 聚合结果，检查完成度和一致性，缺失的部分决定重派还是降级。

伪代码骨架大致是：

```text
plan    = orchestrator.decompose(task)
futures = [spawn(subagent_i, plan[i], tools=allowlist_i)
           for i in tasks if independent]
results = gather(futures, timeout=120s, on_fail=partial_ok)
final   = orchestrator.merge(results)
```

## 踩坑点

- **回传太长**：早期让 subagent 直接返回全文，主 Agent context 瞬间爆炸。改成"摘要 + artifact 路径"后问题消失；
- **并行写同一文件**：两个 subagent 同时改一份配置导致互相覆盖。拆分时必须按资源分区，一个文件只归一个 subagent；
- **没有超时兜底**：一个 subagent 卡死拖垮整批。加上 timeout 和部分结果接受逻辑后，整体可用性明显提升；
- **假并行**：看似独立、实则共享状态的任务（比如都要先跑同一个初始化脚本），并行只会制造竞态。拆之前先画依赖；
- **成本失控**：并行意味着 token 消耗近似翻倍，上量前先在小任务上核算单次成本。

## 可复用建议

1. **契约先行**：先写死 subagent 的输入输出 schema，再写执行逻辑；
2. **幂等设计**：每个子任务可安全重跑，重试才没有心理负担；
3. **并发保守起步**：3~5 足够覆盖大多数场景，先保证稳定性；
4. **独立 trace**：每个 subagent 单独记录日志，排障时能快速定位是哪个分支出的问题；
5. **串行兜底**：并行失败时自动降级为串行执行，宁慢勿挂。

## 总结

subagent 编排本质上不是模型问题，而是工程问题：拆分要真独立、回传要克制、失败要有兜底。模型能力决定上限，编排质量决定下限。建议从"一个主 Agent + 两个只读 subagent"的最小结构跑通全链路，再逐步扩展并发和工具面，比一步到位的复杂编排靠谱得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/e1732983e378d432.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/907a38104b2a51d3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/2170fc40447cde02.png)

