---
title: OpenClaw 多模型路由实战：什么任务给 GPT，什么任务给本地模型
feedId: 40018
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

OpenClaw 里的 agent 任务差别很大：有的是多步规划加 MCP 工具编排，有的只是抽字段、打标签、做摘要。全部走云端 API，账单和延迟都难看；全部压给本地小模型，工具调用又频繁翻车。路由层的意义，就是把不同性质的任务分给合适的模型。

## 问题

我们最早的做法是"插件里随手 if-else"，哪个模型顺手用哪个。三个月后问题集中暴露：

- token 花销没有归因，说不清钱花在哪条链路上；
- 出了质量问题不知道追哪个模型的责任；
- 本地模型在多工具场景下 JSON 参数经常残缺，直接带崩 MCP 调用；
- 路由决策散落在各个插件里，没人能说清全局策略。

## 做法

最后我们收敛成一张显式路由表加配置化 fallback，分五步。

**第一步，按四个维度给任务分类**：是否要求严格的工具调用 / 结构化输出；上下文长度；是否含敏感数据；延迟容忍度。

**第二步，按任务类型映射到三档**：

- A 档（云端强模型）：多步规划、MCP 编排、复杂代码生成；
- B 档（本地中档模型，Qwen 14B 级别）：摘要、分类、格式转换、单工具调用；
- C 档（本地小模型 / embedding）：打标、去重、向量化。

**第三步，落到 OpenClaw 路由配置**，按 agent 角色或任务类型绑定模型 profile，字段名以你用的版本为准：

```yaml
routes:
  - match: task.type == "plan" or task.tool_count > 1
    model: cloud/strong
  - match: task.type == "summarize"
    model: local/qwen-14b
fallback:
  from: local/*
  to: cloud/*
  trigger: schema_validation_failed
  max_retries: 1
```

**第四步，加校验门禁**：本地模型输出先过 JSON Schema / 工具参数校验，不过就升级云端重试一次。

**第五步，观测**：按路由维度记录 token、延迟、失败率，每周复盘一次，路由表当代码 review。

## 踩坑点

1. **本地模型的 tool calling 是重灾区**。14B 以下模型在多工具、嵌套参数场景下 JSON 经常残缺，别把 MCP 编排路由给它们。
2. **上下文静默截断**。本地模型 8k 窗口，长会话直接截头且不报错，路由前先估 token 数。
3. **升级无上限**。fallback 没设 `max_retries`，一次批量任务能把当月预算打穿，预算守卫要和路由同步上线。
4. **"本地更快"是错觉**。GPU 被别的任务占着时，本地排队延迟比云端 API 还高，要给本地推理端点做健康检查。
5. **隐私不能只靠路由**。约定"敏感数据不出内网"之后，上游脱敏仍然必须做，否则一条手滑的路由规则就是事故。

## 可复用建议

- 从两档开始（云端强模型 + 本地一个中档），跑两周再加档位，别一上来建五级路由。
- 路由表放仓库里，改路由 = 提 PR，可 review、可回滚。
- 所有本地输出默认过 schema 校验，这一条挡住了我们大部分的隐性翻车。
- 成本按 route 维度打标归因，否则优化无从谈起。

## 总结

多模型路由本质是在质量、成本、隐私之间做显式取舍。关键不是选哪个模型，而是把决策从代码里捞出来，变成一张可观测、可回滚的表。OpenClaw 的路由能力是够用的，区别只在你愿不愿意把规则写清楚。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/65a1b7fd7addb20a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/3cf814950db95914.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/dd7c296d8986cbe5.png)

