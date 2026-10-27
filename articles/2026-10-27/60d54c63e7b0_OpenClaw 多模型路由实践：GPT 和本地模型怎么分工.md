---
title: OpenClaw 多模型路由实践：GPT 和本地模型怎么分工
feedId: 42768
source: 综合讨论
publishedAt: 2026-10-27
---

# 背景

我们团队用 OpenClaw 跑了一批自动化 Agent：消息分发、报表生成、代码巡检。早期图省事，所有 Agent 都挂在云端大模型上，月底 token 账单不太好看；后来尝试全切本地（Ollama 跑 Qwen），复杂任务的工具调用成功率又明显下滑。多模型路由不是为了炫技，是为了把"哪些环节真的需要强模型"想清楚。

# 问题

OpenClaw 的 Agent 链路里，不同环节对模型能力的要求差异很大：意图识别、槽位抽取这类任务，7B 本地模型足够；多步规划、代码生成、长上下文推理，本地模型就开始频繁翻车。路由策略如果拍脑袋定，要么成本高，要么质量差，而且问题常以"偶发失败"的形式出现，很难定位。

# 做法

1. **先给任务分类。** 把 Agent 链路拆成五类：意图分类、参数抽取、工具编排与规划、代码生成、最终汇总。在 OpenClaw 里给每个子 Agent 打上任务标签。
2. **配置路由规则。** OpenClaw 支持按 Agent 粒度指定模型，我们用确定性规则起步：分类和抽取走本地 7B，规划和代码走 GPT，汇总视上下文长度决定。简化配置如下：

```yaml
router:
  rules:
    - match: task=classify|extract
      model: local/qwen2.5-7b
    - match: task=plan|codegen
      model: gpt-4o
    - default: local/qwen2.5-7b
      escalate_on: [tool_call_failed, low_confidence]
```

3. **加升级策略。** 本地模型工具调用失败或置信度低时，自动升级到云端模型重试一次，并把升级事件写入 trace。
4. **度量再迭代。** 用 OpenClaw 的 trace 回调收集每个任务标签下的成功率、p95 延迟和 token 成本，跑两周后再调整规则边界。

# 踩坑点

- 本地模型上下文窗口小，MCP 工具返回的长 JSON 直接塞进去会截断，导致字段丢失。先做一轮压缩或截取再进模型。
- 本地模型的 tool call 格式错误率高，必须配严格的 schema 校验加有限次重试，否则 Agent 容易陷入循环。
- 升级策略不能只看报错。本地模型经常"沉默失败"——格式正确但内容错误，纯靠 error code 升级会漏掉一截。我们加了个轻量校验 Agent 兜底。
- 同一块 GPU 上跑 embedding 和 LLM，并发上来后本地延迟 p95 飙升，路由看起来"变慢"其实是排队。
- 本地模型一升级（尤其换量化版本），工具调用成功率会悄悄下滑。务必固定模型版本，保留评测集。

# 可复用建议

- 按任务路由，不要按用户或时段路由，规则更稳定、更好排查。
- 先用确定性规则跑稳，再考虑用小模型做路由决策，别一上来就上"智能路由"。
- 保留一个总开关：一键全切 GPT 或全切本地，排查问题时非常有用。
- 每类任务维护 50~100 条 golden case，换模型前后各跑一遍，用数据说话。
- 成本按"完成的任务数"统计而不是按 token，才能和业务对齐。

# 总结

多模型路由本质上是个工程问题：先把链路拆细、任务分类，用最简单的规则跑起来，再用 trace 数据修正边界。GPT 负责真正需要推理和规划的少数环节，本地模型吃掉高频、格式化的多数环节，成本和质量的平衡点自然就出来了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/19843e2642b56856.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/a2ea7c64b7f99be3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/ddd218fba1fcb77b.png)

