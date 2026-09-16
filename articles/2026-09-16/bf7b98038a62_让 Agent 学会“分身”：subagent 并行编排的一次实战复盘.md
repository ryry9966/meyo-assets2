---
title: 让 Agent 学会“分身”：subagent 并行编排的一次实战复盘
feedId: 37798
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

单个 Agent 顺序干活有两个老问题：慢，以及上下文被污染。比如让一个 Agent“调研三个候选 MCP server、各跑一轮 demo、再写对比总结”，它只能一件件来，前面任务的中间输出会挤占上下文，到后面判断质量明显下滑。subagent（子代理）编排是社区里比较成熟的解法：主 Agent 只做拆解和汇总，把独立子任务分发给多个并行的子 Agent。

## 问题

上周我有个真实需求：为内部工具筛选三套方案，各自要查文档、跑 demo、写结论。串行做要四十多分钟，而且第一个方案的长篇调研会污染后面两个方案的判断。这类“子任务相互独立、结果需要汇总”的场景，就是并行 subagent 的甜点区。

## 做法

我的编排结构分四步：

1. **拆任务**。主 Agent 先把需求拆成互不依赖的子任务，每个子任务写清：目标、可用工具、输出格式（固定 JSON 字段，如 `conclusion`、`risks`、`evidence`）。
2. **隔离上下文**。每个 subagent 有独立会话和独立的工具白名单。查文档的只给搜索和抓取工具，跑 demo 的只给执行环境，避免误操作。
3. **并行派发**。主 Agent 同时发出三个 subagent，各自设置超时（如 10 分钟）和 token 预算上限。
4. **汇总裁决**。全部返回（或超时返回部分结果）后，由主 Agent 按统一模板合并成对比表，对互相矛盾的证据标记“待人工确认”。

伪代码大致是：

```text
plan    = main_agent.split(goal, constraints)
futures = [spawn(subagent_i, task, tools, budget) for task in plan]
results = gather(futures, timeout=600)
report  = main_agent.merge(results, schema)
```

## 踩坑点

- **两个 subagent 同时写同一个文件**。第一版我让调研和 demo 两个子任务都能写同一份笔记，结果后写的把先写的结论覆盖了。改成每个 subagent 只写自己的独立文件，由主 Agent 统一合并。
- **不约束输出格式就是灾难**。不指定 schema 时，子 Agent 有人返回散文、有人返回列表，合并脚本直接崩。现在每个 subagent 的 prompt 里强制 JSON schema，解析失败重试一次。
- **并行数不是越多越好**。开到 8 个并行时撞了 API 限流，失败率反而上升。我的经验值是 3~5 个，且每个都带独立的退避重试。
- **失败要显式传递**。子任务超时不能静默吞掉，否则汇总报告会出现“看起来完整、其实缺一块”的结论。超时的子任务在报告里必须显式标注为未完成。

## 可复用建议

- 判断标准：子任务之间没有数据依赖、且单任务耗时超过几分钟，才值得并行；否则编排开销大于收益。
- subagent 的 prompt 里写清楚“不要做的事”，和写清楚“要做的事”同样重要。
- 让主 Agent 只做拆解和汇总，不参与具体执行，能有效控制它的上下文膨胀。
- 给每个 subagent 设硬性的 token 和时间预算，超了就返回部分结果加失败标记。

## 总结

subagent 并行编排不是银弹，它的本质是用结构化的任务拆解，换取速度和上下文清洁度。实践中最值钱的不是“同时开几个”，而是三件事：任务边界是否干净、输出是否结构化、失败是否显式。把这三点做扎实，主 Agent 才能真正当一个省心的包工头，而不是一个被细节淹没的救火队员。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/f892c6ca7b827a19.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/cb7b45225ca4fc2c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/b75bcde647dad3fd.png)

